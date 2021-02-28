---
id: 2178
title: 'Meta Testing: Enforcing MockBehavior.Strict in tests'
date: 2020-02-04T05:14:11+10:00
author: eddiewould
excerpt: 'Testing the tests - a bridge too far?'
layout: post
guid: https://eddiewould.com/?p=2178
permalink: /2020/02/04/meta-testing-enforcing-mockbehavior-strict-in-tests/
categories:
  - Uncategorized
tags:
  - Cecil
  - CIL
  - conventional
  - mockbehavior
  - moq
  - strict
  - unit-testing
---

_…testing the tests… a bridge too far?_

I've recently been introducing the excellent dotnet library <a href="https://github.com/andrewabest/Conventional" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">Conventional</a> into a project at work. For those that haven't encountered it before, Conventional
> Provides a suite of ready-made tests for enforcing conventions within your types, assemblies, solutions and databases to make sure your duckies are all in a row".

This library has proven extremely useful already so naturally I've been seeing what else I can achieve with it. 

When it comes to writing unit tests, I'm a big fan of of the <a href="https://github.com/Moq/moq4/wiki/Quickstart" target="_blank" rel="noreferrer noopener" aria-label=" (opens in a new tab)">Moq</a> library. Like many similar libraries, Moq facilitates the setup of the object-graph for the unit ("system under test") in a couple of ways:

1. Frees you from having to supply an entire object graph of dependencies (A → B → C…)
2. Allows you to control how those dependencies (collaborators) respond to *specific calls*, allowing you to test particular scenarios.

The simplest possible usage is along the lines of

```csharp
// Injected dependency
public interface IResultService {
    string ComputeSomeResult();
}

// System under test
public class MyClass {
    private readonly IResultService _resultService;

    public MyClass(IResultService resultService) {
        _resultService = resultService;
    }

    public string DoFoo() => $"Hello, {_resultService.ComputeSomeResult()};
}

// ...

// Arrange
var resultServiceMock = new Mock<IResultService>();
resultServiceMock.Setup(s => s.ComputeSomeResult()).Returns("world");

var sut = new MyClass(resultServiceMock.Object);

// Act
var result = sut.DoFoo()

// Assert
Assert.Equal("Hello, world", result);
```

When using Moq, I'm an advocate of a couple of practices:

1. Prefer "classic" (state-based) testing (i.e. returned values) rather than interation testing (i.e. `Verify()` calls)
2. Prefer Strict MockBehavior. That is, the mock instance is configured to be _fragile_ and to throw an exception if it receives a call it hasn't been *specifically set up for*.

This post concentrates on the second point (`MockBehavior`). By default, Moq will use `MockBehavior.Loose` (which means any calls that haven't been set up will return default values). This can result in random null reference exceptions down the track but more importantly, *default values for value types* (structs, integers etc). 

Without going into too much detail, the benefits of `MockBehavior.Strict` that I've observed:

1. It helps prevent tests that pass by coincidence (`GetStackDepth()` returns `0`, even through `Push()` has been called twice)
2. It exposes problems in the design (if the collaborator is receiving calls that aren't relevant to what is being tested, I'd like to ask *why*)

To enable strict mock behaviour, one must pass the enum value `MockBehavior.Strict` into the constructor. For example:


```csharp
var resultServiceMock = new Mock<IResultService>(MockBehavior.Strict);
```

Note that there are several constructors for `Mock<,>`

This got me wondering - can I enforce the use of `MockBehavior.Strict` in the unit tests across my project (assuming Moq is being used as the mocking library) with a convention? 

That is to say, if someone writes a unit test that uses `MockBehavior.Loose` (or doesn't specify the MockBehavior) *can we have a failing unit test to report that as a problem*?

With a bit of faffing about, I came up with `MockBehaviorStrictMustBeUsedConventionSpecification` to enforce it. This convention is based on `MustNotUseMethodSpecification` from the <a rel="noreferrer noopener" aria-label=" (opens in a new tab)" href="https://github.com/andrewabest/Conventional/" target="_blank">Conventional source code</a>.

*The basic jist of the approach:*

1. Discover the names of the constructors we're interested in. In practice that's just one constructor (with several overloads)
2. Discover all of the methods *in the type that we're running the convention check against* (including the extra methods created due to the state machines resulting from the use of `async` / `yield`)
3. Iterate through the <a rel="noreferrer noopener" aria-label=" (opens in a new tab)" href="https://en.wikipedia.org/wiki/Common_Intermediate_Language" target="_blank">CIL</a> instructions in those methods, looking for instructions where the OpCode is `NewObj` and the name of the method being called matches one of the constructors we identified in #1. The library <a rel="noreferrer noopener" aria-label=" (opens in a new tab)" href="https://github.com/jbevain/cecil" target="_blank">Mono.Cecil</a> is used to help with this. Along with the instrution, we capture the calling method name.
4. Iterate through those constructor invocations, looking at the (compile time) types of the parameters. Specifically, we're looking to see if there is a parameter with type `MockBehavior` and if so, the _position_ of that parameter.
   * If we don't find such a parameter, the test fails (suitable error is shown to the user)
5. Work backwards from the `NewObj` instruction, looking for the instruction that loaded the value of `0` onto the stack (`MockBehavior.Strict` has an `int32` value of `0`).
   * We use the position of the MockBehavior parameter (relative to the length of the constructor parameter list) to determine how many instructions to go back.
   * If we don't find the instruction `Ldc_I4_0` (load integer value 0) then the test fails (suitable error is shown to the user).
6. Otherwise, test passes - happy camper.

Here's how the convention can be applied (xUnit):

_NB: Helper method to enumerate the test assemblies not shown._


```csharp
[Theory]
[MemberData(nameof(EnumerateTestAssemblies))]
public void Convention_TestsUsingMoqShouldSpecifyMockBehavior(Assembly assembly)
{
    // Require that MockBehavior.Strict be used with Moq (prevent tests passing by coincidence)

    assembly.GetTypes()
        .MustConformTo(new MockBehaviorStrictMustBeUsedConventionSpecification())
        .WithFailureAssertion(ConventionHelpers.Fail);
}
```

The approach seems to work - keen to hear any feedback you may have / alternative approaches for achieving the same thing.

For completeness - the implementation:

```csharp
public class MockBehaviorStrictMustBeUsedConventionSpecification : ConventionSpecification
{
    public override ConventionResult IsSatisfiedBy(Type type)
    {
        var mockCtorNames = new HashSet<(string ModuleName, string MethodName)>(
            typeof(Moq.Mock<>)
                .GetConstructors()
                .Where(c => c.DeclaringType != null)
                .Select(c => (c.DeclaringType.Namespace, c.DeclaringType.Name))
        );

        var methods =
            type.ToTypeDefinition()
                .Methods
                .Where(method => method.HasBody)
                .Select(method => new { Method=method, method.Name })
                .Union(
                    type.ToTypeDefinition()
                        .Methods
                        .Where(containingMethod => containingMethod.HasAttribute<AsyncStateMachineAttribute>())
                        .SelectMany(
                            containingMethod => containingMethod
                                .GetAsyncStateMachineType()
                                .Methods
                                .Where(method => method.HasBody)
                                .Select(method => new {Method=method, containingMethod.Name}))
                ).Union(
                    type.ToTypeDefinition()
                        .Methods
                        .Where(containingMethod => containingMethod.HasAttribute<IteratorStateMachineAttribute>())
                        .SelectMany(
                            containingMethod => containingMethod
                            .GetIteratorStateMachineType()
                                .Methods
                                .Where(method => method.HasBody)
                                .Select(method => new { Method = method, containingMethod.Name }))
                );

        var mockConstructorCalls = new List<(MethodReference MoqConstructorCall, Instruction NewObjInstruction, string ContainingMethodName)>();

        foreach (var methodDetails in methods)
        {
            var instructions = methodDetails.Method.Body.Instructions
                .Where(x => x.OpCode == OpCodes.Newobj && x.Operand is MethodReference method &&
                            mockCtorNames.Contains((method.DeclaringType.Namespace, method.DeclaringType.Name)));

            mockConstructorCalls.AddRange(instructions.Select(instruction => (instruction.Operand as MethodReference, instruction, methodDetails.Name)));
        }

        foreach (var callDetails in mockConstructorCalls)
        {
            // Examine the parameters list to find MockBehavior parameter (and how far it is from the end of the parameters list)
            // If it's the last parameter then 0, second-to-last parameter then 1 and so on.
            int? parameterDistance = null;
            foreach (var (parameter, index) in callDetails.MoqConstructorCall.Parameters.Reverse().Select((value, index) => (value, index)))
            {
                if (parameter.ParameterType.FullName == typeof(Moq.MockBehavior).FullName)
                {
                    parameterDistance = index;
                    break;
                }
            }

            // If we didn't find a MockBehavior parameter then bail.
            if (parameterDistance == null)
            {
                return ConventionResult.NotSatisfied(type.Name, string.Format(FailureMessage, type.Name, callDetails.ContainingMethodName));
            }

            // Starting from the NewObj instruction, work backwards to find the instruction that loads the MockBehavior value onto the stack
            // We go back parameterDistance + 1 instructions
            Instruction instruction = callDetails.NewObjInstruction;

            do
            {
                instruction = instruction.Previous;
                parameterDistance--;
            } while (parameterDistance >= 0);

            // Expecting Ldc_I4_0 (corresponds to MockBehavior.Strict=0) 
            if (!instruction.OpCode.Equals(OpCodes.Ldc_I4_0))
            {
                return ConventionResult.NotSatisfied(type.Name, string.Format(FailureMessage, type.Name, callDetails.ContainingMethodName));
            }
        }

        return ConventionResult.Satisfied(type.FullName);
    }

    protected override string FailureMessage => "{0} uses Moq.Mock without specifying MockBehavior.Strict in method {1}";
}
```

---
id: 2104
title: 'WebAPI + AngularJS form validation &#8211; keeping it DRY'
date: 2019-05-26T14:04:48+10:00
author: eddiewould
layout: post
guid: https://eddiewould.com/?p=2104
permalink: /2019/05/26/webapi-angularjs-form-validation-keeping-it-dry/
categories:
  - Uncategorized
---

Server-side validation that _feels like client-side validation to the user_ w/ ASP.NET & AngularJS

## Introduction

Even though AngularJS is approaching end-of-life, it's still deployed in loads of production systems. This post describes some techniques I used to have the validation logic entirely defined (and performed) server-side (ASP.NET WebAPI) - *whist giving a client-side validation experience*. By that I mean, type into a field and see validation errors as you type (rather than only when you submit the form).

Why the heck would you want to even do that? Well - for one, there are some forms of validation that can _only_ be performed server side - in my case, the example was validating changes to a stored procedure in a SQL database. 

The more prominent reason though would be keeping things "DRY" (don't repeat yourself). *You can't get away from having validation logic defined server side* (remember kids, JavaScript that runs on the client side can <u>easily be tampered with</u>) so why bother *reimplementing* it client-side?

The reason we typically have client-side validation is for a better user experience. See mistakes as you make them, get hints/suggestions, tidying etc.

The approach I took was to not only _define_ the validation logic server-side, but also to exclusively _perform_ it there too. A perhaps more elegant approach would be to *define validation logic in terms of rules* and then automatically generate appropriate code for server-side & client-side validation based on those rules. Anyway - I digress.

## Implementation

Server side, we have a couple of controller actions - one that validates the model and (if successful) persists the changes, the other which just does a dry run (validates the model). Note that in my case the validation involved making stored-procedure changes under a database transaction then *rolling back that transaction* so that the changes weren't actually persisted.

```csharp
[HttpPost]
public IHttpActionResult Validate([FromBody]RuleAndGroupAssignmentsModel fromClient)
{
    var validationResult = ParseAndValidateModel(fromClient, ModelState);
    var response = validationResult.Response;

    if (!ModelState.IsValid)
    {
        return RuleProcessingFailure(response);
    }

    return Ok(response);
}

[HttpPut]
public IHttpActionResult Store([FromBody]RuleAndGroupAssignmentsModel fromClient)
{
    var validationResult = ParseAndValidateModel(fromClient, ModelState);
    var response = validationResult.Response;

    if (!ModelState.IsValid)
    {
        return RuleProcessingFailure(response);
    }

    var rule = RuleViewModelMapper.ToRule(fromClient.Rule, fromClient.GroupAssignments, validationResult.TriggerStoredProc, validationResult.MessageStoredProc);
    int ruleId = _rulesRepository.Upsert(rule);
    response.RuleId = ruleId;
    return Ok(response);
}
```

We also have an extension method to serialize the ASP.NET ModelStateDictionary (errors) into JSON. I kind of half-arsed that - there's some JavaScript code (`parseAspNetModelStateDictionary`) that subsequently turns it into something _actually_ useful (i.e. nested objects). It probably would have  been better to do the transformation entirely server-side.

```csharp
public static class ModelStateDictionaryExtensions
{
    public static IDictionary<string, string> GetErrors(
        this ModelStateDictionary modelState,
        string stripPrefix="")
    {
        var result = new Dictionary<string, string>();

        foreach (var pair in modelState)
        {
            if (pair.Value.Errors.Count > 0)
            {
                var key = pair.Key.StartsWith(stripPrefix) ? pair.Key.Substring(stripPrefix.Length) : pair.Key;
                var value = string.Join("; ", pair.Value.Errors.Select(e => e.ErrorMessage));

                result.Add(key, value);
            }
        }

        return result;
    }
}
```

The AngularJS markup for the form is pretty bog standard - aside from the `server-validation` & `ng-change` directives on the input controls 

```html
<form ng-submit="$ctrl.onSubmit()">
    <div class="form-horizontal">
        <fieldset>
            <legend>Rule details</legend>

            <div class="form-group">
                <label class="control-label col-md-2" for="RuleEnabledCheckbox">
                    Rule enabled
                </label>
                <div class="col-md-10">
                    <input server-validation type="checkbox" ng-model="$ctrl.Rule.Enabled" ng-change="$ctrl.queueRefresh()" id="RuleEnabledCheckbox" />
                </div>
            </div>

            <div class="form-group">
                <label class="control-label col-md-2" for="MessageTitleInput">
                    Message title
                </label>
                <div class="col-md-10">
                    <input server-validation type="text" ng-model="$ctrl.Rule.MessageTitle" ng-change="$ctrl.queueRefresh()" id="MessageTitleInput" class="form-control" />
                </div>
            </div>
            
            ...
            
            <fieldset>
                <legend>Thresholds</legend>
                <table class="table-striped table-hover table-bordered thresholds-table" id="thresholds">
                    <thead>
                        <tr>
                            <th>Parameter name</th>
                            <th>Parameter value</th>
                            <th>Parameter units</th>
                            <th>Parameter SQL data-type</th>
                            <th>Notes</th>
                            <th> </th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr ng-repeat="threshold in $ctrl.Rule.ThresholdValues" ng-init="basePath = 'Rule.ThresholdValues[' + $index + ']'">
                            <td>
                                <input server-validation server-validation-path="{{basePath}}.ParameterName" type="text" ng-model="threshold.ParameterName" ng-change="$ctrl.queueRefresh()" />
                            </td>
                            <td>
                                <input server-validation server-validation-path="{{basePath}}.ParameterValue" type="text" ng-model="threshold.ParameterValue" ng-change="$ctrl.queueRefresh()" />
                            </td>
                            <td>
                                <select server-validation server-validation-path="{{basePath}}.Unit" ng-model="threshold.Unit" ng-change="$ctrl.queueRefresh()" ng-options="option.value as option.name for option in $ctrl.unitOptions"></select>
                            </td>
```

The magic happens in the `server-validation` directive: 

```javascript
const ServerValidationFactory = ($compile) => {
        const errorType = "servervalidation";

        return {
            restrict: "A",
            require: ['ngModel', '^form'],
            link: function(scope, element, attr, controllers) {                
                const [ngModel, form] = controllers;
                let path = attr.ngModel.replace(/\$ctrl\./g, "");
                if (attr.serverValidationPath) {
                    path = attr.serverValidationPath;
                }

                const watchPath = `$ctrl.serverValidityResults.${path}`;

                const validationMessageScope = scope.$new(true);
                const serverValidationMessage = $compile("<server-validation-message />")(validationMessageScope);
                serverValidationMessage.insertAfter(element);

                const updateValidity = (valid) => {
                    ngModel.$setValidity(errorType, valid);
                    validationMessageScope.valid = valid;
                }

                scope.$watch(watchPath, (error) => {
                    validationMessageScope.errorMessage = error;

                    if (error && ngModel.$dirty) {
                        updateValidity(false);
                    } else {
                        updateValidity(true);
                    }
                });

                scope.$watch(`${form.$name}.$submitted`, (attempted) => {
                    if (attempted) {
                        ngModel.$setDirty(true);
                        if (validationMessageScope.errorMessage) {
                            updateValidity(false);
                        }
                    }
                })
            },
        }
    };
```

Note that this directive is restricted to _attribute_ usage since we expect to apply it to `input` (and `textarea` / `select`) elements. It has a couple of key responsibilities:

* To append a `server-validation-message directive` *immediately after* the `<input>` element in question. Note that `server-validation-message` is an element-based directive
* To watch the for server validation messages associated with the input and pass them on to the `server-validation-message` directive. All of the server validation results are expected to be under a property called `serverValidationResults` on the controller instance. By default, the watch path is based on the `ngModel` associated with the input, however we allow it to be specified explicitly with an attribute (`serverValidationPath`). This was necessary when the inputs were used in a `ng-repeat` block. 

The `server-validation-message` directive is so trivial I've only included it for completeness: 

```javascript
const ServerValidationMessageFactory = () => {
        return {
            restrict: "E",
            link: (scope) => {
                scope.valid = true;
            },
            template: "<div ng-if='!valid' style='color: red;'>{{errorMessage}}</div>"
        }
    };
```

Interesting parts of the controller:

* `queueRefresh()` adds a delayed callback to invoke `performRefresh()` after some delay. It does this in a 'debounced' fashion - if we've already set a timer then the existing timer is cancelled and we create a new one.
* `performRefresh()` makes a POST request to the controller. We then deal with the results in `handleRefreshResponse()`
* `handleRefreshResponse()` first checks that the form data hasn't changed since the validation started (if it has, it bails). As well as "tidying" the client model, the primary thing it does is set the `serverValidityResults` property on the controller. Remember, this is the property which the `server-validation` directives watch under. There also some code to parse the JSON representation of the `ModelStateDictionary` errors into something more useful:

```javascript
queueRefresh = () => {
    if (this.refreshTimeout) {
        this.$timeout.cancel(this.refreshTimeout);
        delete this.refreshTimeout;
    }

    this.refreshTimeout = this.$timeout(() => {
        this.performRefresh();
    }, this.REFRESH_DELAY);
};

performRefresh() {
    const refreshPayload = this.getPayload();

    const refreshJsonSent = JSON.stringify(refreshPayload);

    const refreshPromise = this.$http.post("/api/rules/validate", refreshPayload).then(response => {
        delete this.refreshTimeout;
        this.handleRefreshResponse(response, refreshJsonSent);
    }, (response) => {
        if (response.status !== 400) {
            window.toastr.error("Validation failed due to an unexpected error");
        }
        delete this.refreshTimeout;
        this.handleRefreshResponse(response, refreshJsonSent);
    });

    this.tracker.track(refreshPromise);
}

handleRefreshResponse(response, jsonSent) {
    // Update the model client side based on server tidying. But only if nothing has changed since
    // we sent the request.
    const currentJson = JSON.stringify(this.getPayload());
    if (currentJson === jsonSent) {
        const data = response.data;

        // Take a little bit of care updating the Rule to avoid unwanted scope changes
        // If the user is typing in one of the threshold input fields, don't disrupt them
        for (const key in data.Model.Rule) {
            if (!data.Model.Rule.hasOwnProperty(key)) {
                continue;
            }

            if (key === "ThresholdValues") {
                this.mergeArrayValues(data.Model.Rule[key], this.Rule[key]);
            } else {
                this.Rule[key] = data.Model.Rule[key];
            }
        }

        this.GroupAssignments = data.Model.GroupAssignments;
        this.serverValidityResults = this.parseAspNetModelStateDictionary(data.Errors);        
    }
}

parseAspNetModelStateDictionary(dict) {
    const result = {};

    const partsRe = /([a-zA-Z0-9]*\[[0-9]+\])|(\.)/g;
    const indexerRe = /^([a-zA-Z0-9]*)\[([0-9]+)\]$/;

    for (const key in dict) {
        if (!dict.hasOwnProperty(key)) {
            continue;
        }

        const value = dict[key];

        const keyParts = key.split(partsRe).filter(p => !!p && p !== ".");
        const finalKeyPart = keyParts.splice(keyParts.length - 1, 1)[0];

        let container = result;
        for (const part of keyParts) {
            if (indexerRe.test(part)) {
                const match = indexerRe.exec(part);
                const arrayName = match[1];
                const arrayIndex = match[2];

                container[arrayName] = container[arrayName] || [];
                const elem = {};
                container[arrayName][arrayIndex] = elem;
                container = elem;
            } else {
                container = container[part] = container[part] || {};
            }
        }

        if (indexerRe.test(finalKeyPart)) {
            const match = indexerRe.exec(finalKeyPart);
            const arrayName = match[1];
            const arrayIndex = match[2];

            const array = container[arrayName] = container[arrayName] || [];
            array[arrayIndex] = value;
        } else {
            container[finalKeyPart] = value;
        }
    }

    return result;
}

mergeArrayValues(sourceArray, targetArray) {
    if (targetArray.length !== sourceArray.length) {
        targetArray.length = 0;
        for (const item of sourceArray) {
            targetArray.push(item);
        }
    } else {
        for (var i=0; i<sourceArray.length; i++) {
            const sourceItem = sourceArray[i];
            const targetItem = targetArray[i];

            for (const key in sourceItem) {
                if (sourceItem.hasOwnProperty(key)) {
                    continue;
                }

                targetItem[key] = sourceItem[key];
            }
        }
    }
}
```

## Summary

This is clearly a lot of trouble to go to just to avoid duplicating some validation logic. Is it worth it? That depends. In simple cases I'd probably say just duplicate the validation logic and live with it. 

Cases where I see it being useful:

* Very large number of form fields
* Complex validation rules
* Dependencies between form-fields with respect to validation
* Validation logic that can only be performed server side (but there are other, much simpler ways to achieve that).

It was a useful learning exercise none-the-less! Hope you find it useful
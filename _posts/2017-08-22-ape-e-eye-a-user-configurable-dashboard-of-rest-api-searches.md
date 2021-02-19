---
id: 14
title: 'Ape E Eye: A user-configurable dashboard of REST API &#8220;searches&#8221;'
date: 2017-08-22T11:42:00+10:00
author: eddiewould
layout: post
guid: http://eddiewould.com/2017/08/22/ape-e-eye-a-user-configurable-dashboard-of-rest-api-searches/
permalink: /2017/08/22/ape-e-eye-a-user-configurable-dashboard-of-rest-api-searches/
restapi_import_id:
  - 5be77bc7d1246
ocean_gallery_link_images:
  - 'off'
ocean_sidebar:
  - "0"
ocean_second_sidebar:
  - "0"
ocean_disable_margins:
  - enable
ocean_display_top_bar:
  - default
ocean_display_header:
  - default
ocean_center_header_left_menu:
  - "0"
ocean_custom_header_template:
  - "0"
ocean_header_custom_menu:
  - "0"
ocean_menu_typo_font_family:
  - "0"
ocean_disable_title:
  - default
ocean_disable_heading:
  - default
ocean_disable_breadcrumbs:
  - default
ocean_display_footer_widgets:
  - default
ocean_display_footer_bottom:
  - default
ocean_custom_footer_template:
  - "0"
ocean_link_format_target:
  - self
ocean_quote_format_link:
  - post
categories:
  - Uncategorized
---

Ape E Eye (<a href="http://ape-e-eye.herokuapp.com/">http://ape-e-eye.herokuapp.com/</a>) is a toy project I built to learn Angular JS. It's a configurable dashboard of "searches" (REST API queries) which are periodically updated. 

It's hosted on GitHub at <a href="http://github.com/flakey-bit/ape-e-eye">http://github.com/flakey-bit/ape-e-eye </a>

This is what it looks like: 

<figure><img src="/wp-content/uploads/2018/11/290be-ape-e-eye.png" class="wp-image-15"/></figure>

The idea is the user sets up one or more searches (REST API calls) and the dashboard will periodically poll that search and display the results. The user must supply the API URL as well as a JavaScript function definition of a "data handler" (to parse/post-process the API result for display).

<figure class="wp-block-image"><img src="/wp-content/uploads/2018/11/fc502-screen2bshot2b2017-08-222bat2b9-34-182bpm.png" alt="" class="wp-image-22"/></figure>

<figure class="wp-block-image"><img src="/wp-content/uploads/2018/11/2492d-screen2bshot2b2017-08-222bat2b9-35-092bpm.png" alt="" class="wp-image-23"/></figure>

It comes with some demo searches, but it's probably more fun if you set-up your own :)

It's built with

* Angular1.6 (with components where applicable)
* NPM for managing external dependencies
* FontAwesome for awesomeness
* Webpack for asset management
* A mixture of TypeScript and Vanilla JS (just to try both approaches)
* Karma for unit tests

Other than interaction with external APIs, there is no server-side functionality at all!

It doesn't make use of Bootstrap or other UI frameworks (though I used CodeMirror for the editor) - it's a pet project so fun to implement things from scratch 😛

There's a FAQ (convering questions like "Why did you use AngularJS when there's x/y/z out there") in the README.md on Github.

Have at it Internet (keen on any feedback, particular on AngularJS best-practices I've ignored!)
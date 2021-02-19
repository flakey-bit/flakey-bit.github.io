---
id: 12
title: 'Loading G-NAF (Geocoded National Address File) into SQL Server &#038; making a simple query'
date: 2018-08-30T05:44:00+10:00
author: eddiewould
layout: post
guid: http://eddiewould.com/2018/08/30/loading-g-naf-geocoded-national-address-file-into-sql-server-making-a-simple-query/
permalink: /2018/08/30/loading-g-naf-geocoded-national-address-file-into-sql-server-making-a-simple-query/
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

I had a few hours spare and so thought I'd have a play with the "G-NAF" data set provided <b>free of charge</b> by the Australian government

### What is G-NAF?

G-NAF (Geocoded National Address File) is a trusted index of Australian address information. It contains the state, suburb, street, number and coordinate reference (or “geocode”) for street addresses in Australia.

It's worth noting that it doesn't contain personal or business related data - just data on physical addresses in Australia.

It comes as a ZIP file containing SQL scripts to define the database schema as well as pipe-delimited data files. More information can be found <a href="https://www.psma.com.au/sites/default/files/g-naf_product_description.pdf" target="_blank" rel="noopener noreferrer">here</a>

### Where can I get it?

Head to <a href="https://data.gov.au/dataset/geocoded-national-address-file-g-naf" target="_blank" rel="noopener noreferrer">https://data.gov.au/dataset/geocoded-national-address-file-g-naf</a>

<figure class="wp-block-image"><img src="/wp-content/uploads/2018/11/d8484-go-to-resource.png" alt=""/></figure>

Click the link "Go to resource" as shown above.

### How do I get it into SQL server?

These are the steps I followed - there's more than one way to skin a cat, as they say.


1. Unzip the archive e.g. `aug18_gnaf_pipeseparatedvalue_20180827115521.zip` into a suitable location (e.g. c:\source)
2. Install SQL Server (I used the "Developer" edition 2017; "Express" probably would have been fine also.
3. Create a new database called GNAF. I used the default collation (SQL_Latin1_General_CP1_CI_AS) and that seems to have worked fine.
4. Put your database in simple recovery to speed things up: `ALTER DATABASE GNAF SET RECOVERY SIMPLE;`
5. Execute the script `G-NAF\Extras\create_tables_sqlserver.sql` against the GNAF database (I ran the script via SQL Server Management Studio)
6. Import data - I made use of a couple of PowerShell scripts to load the data in via BCP (hosted on gist.github.com)
  * <a rel="noopener noreferrer" href="https://gist.github.com/flakey-bit/a9a358907a39ac47f2f38da5223e86a5" target="_blank">load_authority_codes.ps1</a>
  ```powershell
  PS .\load_authority_codes.ps1 -serverName eddie-pc -databaseName GNAF -authorityCodeFilesPath '.\AUG18_GNAF_PipeSeparatedValue_20180827115521\G-NAF\G-NAF AUGUST 2018\Authority Code'
  ```
  * <a rel="noopener noreferrer" href="https://gist.github.com/flakey-bit/87ec2eaae045d14db77611674aaa1fa9" target="_blank">load_jurisdiction_codes.ps1</a>
  ```powershell
  PS .\load_jurisdiction_data.ps1 -serverName eddie-pc -databaseName GNAF -jurisdictionFilesPath '.\AUG18_GNAF_PipeSeparatedValue_20180827115521\G-NAF\G-NAF AUGUST 2018\Standard' -jurisdictionName QLD
  ```  
7. Create the "sample" view by executing G-NAF\Extras\GNAF_View_Scripts\address_view.sql (I ran the script using SQL Server Management Studio)
8. _Optionally_, create foreign key constraints by executing `G-NAF\Extras\GNAF_TableCreation_Scripts\add_fk_constraints.sql` (I didn't bother)

It only took a few minutes to import the data for Queensland and roughly the same for Western Australia (inside a VM).

### What can I do with it?

Well I'm still figuring that part out. You can run simple "geocoding" type queries as below:

<figure class="wp-block-image"><img src="/wp-content/uploads/2018/11/geocoding-sql-query.png" alt="geocoding-sql-query" class="wp-image-30"/></figure>

<figure class="wp-block-image"><img src="/wp-content/uploads/2018/11/readify-geocoded.png" alt="readify-geocoded.png" class="wp-image-31"/></figure>

Have fun with it!
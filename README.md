##Mapping Conflict in the DRC


This documentation is divded into two parts. The first section covers how to set up github and fork the project, and explains the site architecture with file descriptions. The second section backs up to data processing and map design by providing links to tools to download, and docs for working with SQLite and Tilemill. 

##Github and Site Architecture

###Setting up Github
 - [Create an account](https://github.com/signup/free)
 - [Set up Git](http://help.github.com/mac-set-up-git/)
 - [Forking a repository](http://help.github.com/fork-a-repo/)


### Forking the project
To use this project as a starting point, [fork the repository](http://help.github.com/fork-a-repo). This will create a new copy of the project on github. Make changes by setting up a local repo by cloning it to the file or directory that you desire, and then from there you can make changes and commit/push them to github. For example, I have a local github folder that I navigate to with `cd Documents/github/` to clone all github projects. Then i navigate into the spcecific project to commit and push changes. 

###Files
-----

    _includes/     Javascript and css files.
    _layouts/      Major page templates.
    _posts/        Content. Data, sources, about.
    _site/         The static site generated by Jekyll.
    data/          Csv and json files of data for drawer.
    fonts/         Fonts.
    images/        Images.
    tilemill/      Internews-media tilemill project.
    404.html       Custom 404 page - only works with gh-pages.
    CNAME          CNAME record. See http://pages.github.com under 'Custom domains' to see how this works with github pages
    README.md      This file.
    _config.yml    Jekyll configuration file.
    404.html       Custom 404 page - only works with gh-pages.
    import.rb      An import script designed to split the data from csv into indivdual .json files per province
    index.html     Home page.
    site.css       Aggregated site css.
    site.js        Aggregated site scripts.


###Jekyll
The site uses Jekyll, a simple, logic aware, static site generator. In a nutshell you build a site with all the logic and source files you need and Jekyll creates a static copy of the website in a '_site/' directory. Note that you do not touch the contents of this directory. You make any additions or changes to the files outside of this.

Make sure you have [Xcode](https://developer.apple.com/technologies/tools/) installed before installing Jekyll. This is available for free in the Apple App store.  Once this is complete you should be able to run: 

- `gem install jekyll`

If this doesn't work, read documentation [available here](https://github.com/mojombo/jekyll/wiki/install) about updating your ruby packages.

Once jekyll is installed, and you have downloaded the project, from terminal, navigate into the directory the project is in and type 'jekyll'. Locally, the site can be viewed at `http://0.0.0.0:4000/index.html` in any browser after it has been generated for the first time. This allows you to make local changes that are automatically reflected in your local version of the site, as long as jekyll is running in your terminal. To stop jekyll, type 'command c'.

####Site Configuration
`_config.yml` sets up the default configuration for jekyll when reading all files. As seen below, it allows you to format the url, and allows for the exclusion of unnecessary files that jekyll doesn't need to generate.

    auto: true
    server: true
    exclude:
      - README.md
      - import.rb
      - tilemill
    permalink: /:title
    baseurl: /internews-media

####Adding new pages
Jekyll allows for easy referencing and adding logic for other pages in the header of each new page that you create. For our use case, in `0200-01-02-about.html`, the header includes several options:

    title: About the Data
    category: about
    tags: about
    layout: pages

We reference posts with 'tags' and 'category' to control their display from the template pages located in '_layouts/'. The logic contained in these template files uses Liquid. You can learn more about the [Liquid Templating System](https://github.com/shopify/liquid/wiki/liquid-for-designers) for more information on creating relationships between files in the `_site` directory.

####Naming conventions
Jekyll has a default chronological pagination system. Posts are ordered such that the most *recent* post appears first, so we created fake dates to ensure hierarchy between the 'About' and 'Sources' links in the _posts directory.

    hierarchy    file name
    --------     ---------
     1           0200-01-02-about.html
     2           0200-01-01-data.html
     2         0200-01-01-sources.html

###Notes
Where something requires explanation there are inline notes in the code.

##Data Processing
[This tutorial](http://mapbox.com/tilemill/docs/tutorials/SQLite-work/) walks through turning data sources into SQLite files. This requires downloading [Quantum GIS](www.qgis.org), and [Tilemill](www.tilemill.com).

Qgis is a powerful tool for working with geographic files, but for right now we just want it to convert data formats like csvs and shapefiles into SQLite databases. 

SQLite databases are the best tool for sorting data in Tilemill. SQLite lets you change the query on the data whenever you like, to find a different angle on your data. It is also an easy way to join databases with geographic information to those without geographic information, in the `attach db` field of the `add SQLite layer` in Tilemill. This process is outlined in the same [tutorial](http://mapbox.com/tilemill/docs/tutorials/SQLite-work/) as above.

For example, with the LRA attacks and security incidents against humanitarian workers, I saved the original files as csvs, then imported them into QGIS. Then I saved the layers as SQLite databases. 

Since I want to aggregate my LRA incidents by the district level, I need points that correspond to each district. So I looked at the admin2 shapefile from the `cod_boundaries`, which has district information.  In QGIS I add a vector layer and open up this `admin2.shp` file. This opens up polygons, but I don't want to join my LRA or security incidents to polygons, since I was display them as events by location. 

To turn the polygons into centroids, I went to `vector, geometry tools, polygons to centroids.' Then I right-clicked on the shapefile and saved the layer as a SQLite database, and set the CRS (map projection) to google mercator, since this is what Tilemill by default uses. 

Now you should have two SQLite databases, one for LRA attacks, and one for admin2 centroids. In Tilemill you will go to `add a layer`, choose `SQLite` and open up the admin2 centroids file. Then in the `attach DB` field, you will navigate to your LRA attacks file. The join between these two databases happens in the query field, and looks similar to this: 

            (select *,
  
      count(unique_id) as num_attacks, 
      sum(ppl_killed) as ppl_killed, 
      sum(infants_kidnapped) as infants_kidnapped,  
      sum(adults_kidnapped) as adults_kidnapped

        from  `admin3_centroids` a join `lra_attack` b
        on b.territory=a.nom
        group by month, territory
      )
      
The asterisk means select all, and the following rows are counting or adding data so that as we aggregate by district, we won't lose individual information about each attack. This yield a table with a row for each territory of each month in the data with aggregated attacks, people killed, and people kidnapped. 

###Group concatenation 

It's important in the query above to include a [group_concat](http://www.sqlite.org/lang_aggfunc.html) function which will retain the data for each row (representing one event) even though we're aggregating the data by province and month with `group by` statements. 

This is done with a `group_concat` line after your `select` statement. You add in quotation marks the characters that you wish to exist between each concenated item. In our case, we want to build this into a table, so we put table html inbetween each item, and each column that we're concatenating is separated by pipes `||`. Finally, we have to finish the `group_concat` with a `,   ' ') `, to avoid trailing spaces. 

Here's an example of a query with a group_concat command for the sec2012 data: 

`(select b.month, a.GEOMETRY, a.OGC_FID, 
count(unique_id) as num_events,
sum(num_deaths) as num_deaths,
sum(num_injured) as num_injured,
group_concat ('<tr><td>' || date_incident  ||'</td><td>' || locality  || ' </td><td>' ||  type_incident || '</td><td>' || num_injured || '</td><td>' ||  num_deaths || '</td></tr>',   ' ') as drc_interactivity
from admin3_centroids a join sec2012 b
on a.nom=b.territory
group by month,territory)`

This outputs a column in your resulting data that will have the concenated information from each row that was aggregated on month, and territory. We can now reference this column in our tooltips by wrapping it in the rest of hte HTML that makes up a table. As seen here: 

`<table>
<th>Date</th>
<th>Location</th>
<th>Reason</th>
<th>Casualties</th>
<th>Injured</th>

{{{drc_interactivity}}}
</table>
`


##Map Design 

This comes much easier, as it follows a css-type like language called carto. All of the basics and more advanced options of styling your data can be found in the [mapbox.com/help](http://mapbox.com/help), starting with [styling data](http://mapbox.com/tilemill/docs/crashcourse/styling/) section.

##Google image chart Graphs

The charts that show up on the side bar in the site are created in the [google image chart wizard](https://developers.google.com/chart/image/docs/chart_wizard), which allows you to change the margins, styling and size of different kinds of charts. 

However you can easily recreate the charts in the DRC site with the same styling by changing the URLs inserted in the <img> tags within <div='graphs'>. These are tagged with <class='year2010'> or <class='2011'> etc., which is read by the javascript to know when to appear with the year selector. See below:

![](https://img.skitch.com/20120406-d6qh7aff173a16ktkun745xer1.jpg)

These three image tags result in three unique graphs. The data and parameters are defined by the different tags within the URL, and separated by the `&` symbol. Below are the three parameters that must be changed in order to create a new URL for a graph. More information about the other components of these charts can [be found here](https://developers.google.com/chart/image/docs/chart_params).
 
 - the title, `chtt=`
 - the data, `chd=t:`  the different data sets represented here are seperated by a pipe `|`
 - the img class (to match the year of the data you're inserting)
 
![](https://img.skitch.com/20120406-x3715g5spckk9utppr4239khb6.jpg)


##Changing the Returnee and IDP Totals

![](https://img.skitch.com/20120509-jupetxn7ch2sh9whw91i2fc67k.jpg)

The totals are just like the Google image charts, which have classes to determine what year they appear on, for example `<class='year2010'>` or `<class='2011'>`. 

It's also essential to add `style="display: none;` so that all of the numbers don't display on page load. This can be seen below from the file `map-interactive.html`, with arrows pointing to where you add the new numbers. These numbers were collected by running a query on the appropriate IDP and Returnee files to add the numbers for a given year. 

![](https://img.skitch.com/20120509-fgc4j8gkktks2kh2bhmbym2yae.jpg)


##Updating JSON Files to Populate the Tables

To update the tables and maps for new data, such as 2012 data, you need to start by adding a the appropriate option to the year selector from the "map-interactive.html" file. The new line is in bold, and for the map to load with 2012 selected, you need to move the 'selected' parameter to 2012 as well. 

 <select id='year-select'>
      <option value='2011'>2011</option>
      <option value='2010'>2010</option>
      **<option value='2012 selected='selected''>2012</option>**
    </select>
    
Then you need to change the default year in the site.js file to 2012 instead of 2011 on [line 21](https://github.com/RWMaps/rw-drc/blob/gh-pages/_includes/js/site.js#L21).

**var year = year || '2012';**

In the `layers.js` file, add 2012 data (pointing to the 2012 layers), which will directly inform which JSON files are searched for to fill the table. Here is an example of the new [layers.js](https://gist.github.com/1e0a30a59a514a1069a2) file with new values for 2012. These map layers don't exist yet, but it is what they should be named once created.  

Lastly, you need to create the new JSON files. This can be done by updating the data, "idp.csv, sec.csv, lra.csv, and ret.csv" files and then running the import.rb script. Here is a [brief tutorial](http://ruby.about.com/od/tutorials/a/commandline_2.htm) on running ruby scripts in your command line. The script was custom written to import the specific formats of each csv and turn them into JSONs, so it is essential that the new data matches the original format of each csv. 

The tables will only populate if the data layers are appropriately filled out in `layers.js` with a corresponding valid JSON in `data/JSON` for each layer. If even one of these JSONs is missing or invalid, the whole table will break, even though the map still loads any layers. 

The other option is to manually create JSONs for each month for 2012, by copying and pasting the format for each type (lra, idp, sec, ret) into a new file, and changing the values. For example, you can create a new file in the data/json directory called idp-jan-12.json, and fill it with new values, so long as this matches the same format set up in the 2011 JSONs.

##Updating the world map baselayer

The world map is originally set with a baselayer ([releifweb.africa](https://tiles.mapbox.com/reliefweb/map/africa)) and a borders layer ([reliefweb.drc-borders](https://tiles.mapbox.com/reliefweb/map/drc-borders)). These can both be easily changed in the [site.js](https://github.com/RWMaps/rw-drc/blob/gh-pages/_includes/js/site.js#L26) file

Note that the two layers are separated by a comma, if you upload one baselayer that includes both the base map and borders, you will only need one map layer with no ending commas. 

Use the MapBox [create a map](http://mapbox.com/help/#creating_a_new_map) feature to create a custom world base map that can have terrain, labels turned on or off, and any color for the ocean and land. Then when you are done, click save, and then embed. This will pull up a window that has the tilejson url that you need, such as mine seen below. However the reliefweb account will look like reliefweb.map-xxxxxx.

![](https://img.skitch.com/20121004-xdq78jewf5kadrdy98e3fcyp78.jpg). 

This tilejson url replaces the current layers in the site.js file. 

The custom layer made by Relief Web can be combined with an custom MapBox basemap that just has the ocean, as done here in [this example](http://d.tiles.mapbox.com/v3/djohnson.map-m9l4eaq3,reliefweb.1_AVMU_World.html#5.00/10.674/34.673). 


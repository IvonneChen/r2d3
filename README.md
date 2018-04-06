R to D3 rendering tools
================

<img src="tools/README/r2d3-hex.svg" width=200 align="right"/>

`r2d3` renders [D3](https://d3js.org/) scripts that it can be used from the R console, [R Markdown](https://rmarkdown.rstudio.com/), or [Shiny](https://shiny.rstudio.com) like any other [htmlwidget](https://www.htmlwidgets.org/). Specifically, with `r2d3` you can:

-   Render [D3](https://d3js.org/) scripts with ease in R as [htmlwidgets](https://www.htmlwidgets.org/).
-   Use [Shiny](http://shiny.rstudio.com/) with `r2d3` to create interactive D3 applications.
-   Use D3 chunks in [R Markdown](https://rmarkdown.rstudio.com/) and [R notebooks](https://rmarkdown.rstudio.com/r_notebooks.html).

Installation
------------

Install this package by running:

``` r
devtools::install_github("rstudio/r2d3")
```

Getting Started
---------------

To render D3 scripts, `r2d3` provides a `r2d3` JavaScript object that should be used to to retrieve rendering properties:

-   **r2d3.data**: The R data converted to JavaScript.
-   **r2d3.svg**: The svg element with the right dimensions.
-   **r2d3.width**: The width of the svg.
-   **r2d3.height**: The height of the svg.
-   **r2d3.options**: Additional options provided from R.

The `r2d3` object can then be used in a D3 script as follows:

    r2d3.svg.selectAll('rect')
        .data(r2d3.data)
      .enter()
        .append('rect')
          .attr('width', function(d) { return d * 10; })
          .attr('height', '20px')
          .attr('y', function(d, i) { return i * 22; })
          .attr('fill', 'steelblue');

Finally, the above `barchart.js` script can be rendered from R by calling `r2d3` with the data and D3 script to be rendered:

``` r
library(r2d3)
r2d3(
  c(10, 30, 40, 35, 20, 10),
  "barchart.js"
)
```

![](tools/README/barchart-1.png)

Advanced Rendering
------------------

More advanced scripts can rely can make use of `r2d3.onRender()` which is similar to `d3.csv()`, `d3.json()`, and other D3 data loading libraries, to trigger specific code during render and use the rest of the code as initialization code, for instace:

    // Initialization
    r2d3.svg.attr("font-family", "sans-serif")
      .attr("font-size", "8")
      .attr("text-anchor", "middle");
        
    var pack = d3.pack()
      .size([r2d3.width, r2d3.height])
      .padding(1.5);
        
    var format = d3.format(",d");
    var color = d3.scaleOrdinal(d3.schemeCategory20c);

    // Rendering
    r2d3.onRender(function() {
      var root = d3.hierarchy({children: r2d3.data})
        .sum(function(d) { return d.value; })
        .each(function(d) {
          if (id = d.data.id) {
            var id, i = id.lastIndexOf(".");
            d.id = id;
            d.package = id.slice(0, i);
            d.class = id.slice(i + 1);
          }
        });

      var node = r2d3.svg.selectAll(".node")
        .data(pack(root).leaves())
        .enter().append("g")
          .attr("class", "node")
          .attr("transform", function(d) { return "translate(" + d.x + "," + d.y + ")"; });

      node.append("circle")
          .attr("id", function(d) { return d.id; })
          .attr("r", function(d) { return d.r; })
          .style("fill", function(d) { return color(d.package); });

      node.append("clipPath")
          .attr("id", function(d) { return "clip-" + d.id; })
        .append("use")
          .attr("xlink:href", function(d) { return "#" + d.id; });

      node.append("text")
          .attr("clip-path", function(d) { return "url(#clip-" + d.id + ")"; })
        .selectAll("tspan")
        .data(function(d) { return d.class.split(/(?=[A-Z][^A-Z])/g); })
        .enter().append("tspan")
          .attr("x", 0)
          .attr("y", function(d, i, nodes) { return 13 + (i - nodes.length / 2 - 0.5) * 10; })
          .text(function(d) { return d; });

      node.append("title")
          .text(function(d) { return d.id + "\n" + format(d.value); });
    });

``` r
flares <- read.csv("inst/samples/bubbles/flare.csv")
r2d3(
  flares[!is.na(flares$value), ],
  "bubbles.js",
  version = 4
)
```

![](tools/README/unnamed-chunk-6-1.png)

R Markdown
----------

R Markdown can be used with `r2d3` to render a D3 script as an htmlwidget as follows:

<pre><code>---
output: html_document
---

&#96``{r}
library(r2d3)

r2d3(
  c(10, 20, 30),
  "barchart.js"
)

&#96``</code></pre>
For `rmarkdown` documents and Notebooks, `r2d3` also adds support for `d3` chunk that can be use to make the D3 code more readable:

<pre><code>---
output: html_document
---

&#96``{r setup}
library(r2d3)
bars <- c(10, 20, 30)
&#96``

&#96``{d3 data=bars, options='orange'}
r2d3.svg.selectAll('rect')
    .data(r2d3.data)
  .enter()
    .append('rect')
      .attr('width', function(d) { return d * 10; })
      .attr('height', '20px')
      .attr('y', function(d, i) { return i * 22; })
      .attr('fill', r2d3.options);
&#96``</code></pre>
![](tools/README/rmarkdown-1.png)

Shiny
-----

`r2d3` provides `renderD3()` and `d3Output()` to render under Shiny apps:

``` r
library(shiny)
library(r2d3)

ui <- fluidPage(
  inputPanel(
    sliderInput("bar_max", label = "Max:",
      min = 10, max = 110, value = 10, step = 20)
  ),
  d3Output("d3")
)

server <- function(input, output) {
  output$d3 <- renderD3({
    r2d3(
      floor(runif(5, 5, input$bar_max)),
      system.file("baranims.js", package = "r2d3")
    )
  })
}

shinyApp(ui = ui, server = server)
```

We can also render D3 in a Shiny document as follows:

<pre><code>---
runtime: shiny
output: html_document
---

&#96``{r setup}
library(r2d3)
&#96``

&#96``{r echo=FALSE}
inputPanel(
  sliderInput("bar_max", label = "Max:",
    min = 10, max = 110, value = 10, step = 20)
)

bars <- reactive({
   floor(runif(5, 5, input$bar_max))
})
&#96``

&#96``{d3 data=bars}
var bars = r2d3.svg.selectAll('rect')
    .data(r2d3.data);
    
bars.enter()
    .append('rect')
      .attr('width', function(d) { return d * 10; })
      .attr('height', '20px')
      .attr('y', function(d, i) { return i * 22; })
      .attr('fill', 'steelblue');

bars.exit().remove();

bars.transition()
  .duration(250)
  .attr("width", function(d) { return d * 10; });
&#96``</code></pre>
<img src="tools/README/baranim-1.gif" width=550 align="left"/>

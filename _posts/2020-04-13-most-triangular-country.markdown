---
layout: post
title:  "What is the Most Triangular Country?"
date:   2020-04-13 18:12:00 +0100
categories: python
permalink: "/most-triangular-country"
---
# Summary

A list of 196 countries ordered by "triangularity" can be found in the Results section of this post.

Details of the methods are described in the following two sections. I have included all my code at the very end of this post.

I was inspired to write this blog post after reading two excellent posts. One of these is about finding the most rectangular country and is by David Barry. The other is by Gonzalo Ciruelos and is about finding the roundest country. I'll provide links to these posts below:

* [Estimating the rectangularness of countries](https://pappubahry.com/misc/rectangles/code/)
* [What is the roundest country?](https://gciruelos.com/what-is-the-roundest-country.html)

I thought it would be interesting to have a go at calculating the most triangular country. I followed a similar method to the previous posts. The first step is to define mathematically what "triangularity" means. The next step is to find a way to calculate triangularity for every country in the world. 

In summary, the method involves picking a similarity measure between a triangle and a country, then, for each country, finding the triangle that maximises the similarity measure. This maximum similarity is the country's triangularity. Finding a triangle that maximises similarity is an optimisation problem. A triangle can be uniquely defined by the coordinates of its three vertices. Hence, there are 6 free parameters in the optimisation problem (an $$x$$ and $$y$$ coordinate per vertex). In contrast, defining a rectangle requires four parameters, and a circle requires only three (e.g. the coordinates of the center of the circle and its radius). As a result of the greater number of free parameters, this optimisation is more prone to get stuck in local optima. I tackled the problem by running the optimisation multiple times per country with randomised initial conditions.

Overall, I am happy with the results.

# Theory

Following Gonzalo Ciruelos' example, I will define the similarity of two sets $$A$$ and $$B$$ in $$ \mathbb{R}^{2} $$ as the area of the intersection of the sets divided by the area of the larger set.

$$
\textrm{similarity} (A,B) = \frac{\textrm{area}( A  	\cap B)}{\max \{ \textrm{area} {(A)},\textrm{area} (B) \} } 
$$

This definition of similarity has the following properties:

* Because area is never negative, $$\textrm{similarity}(A,B) \geq 0$$.

* Because $$\textrm{area}(A \cap B) \leq  \max \{\textrm{area}(A), \textrm{area}(B)\}$$, $$\textrm{similarity} (A,B) \leq 1 $$.

* If $$A=B$$, then $$\textrm{similarity} (A,B) = 1$$.

* If $$A \cap B = 0$$, then $$\textrm{similarity} (A,B) = 0$$.

(An alternative measure of similarity would be the [Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index). My intuition is that it will produce similar results.)

Let the country $$C$$ be a set in the plane, $$ C \subset \mathbb{R}^{2} $$. Let $$ T(\textbf{x},\textbf{y}) $$ be a set in the plane, $$ T(\textbf{x},\textbf{y}) \subset \mathbb{R}^{2} $$, that is enclosed by the triangle with vertices: $$(x_{1},y_{1})$$, $$(x_{2},y_{2})$$, $$(x_{3},y_{3})$$. 

Then, the triangularity of $$C$$ is the maximum similarity between $$C$$ and some $$ T(\textbf{x},\textbf{y})$$: 

$$
\textrm{triangularity} (C) = \max_{\textbf{x},\textbf{y}}  \textrm{similarity} (C,T(\textbf{x},\textbf{y}))
$$

where

$$\textbf{x} = [x_{1},x_{2},x_{3}]$$

$$x_{i} \in \mathbb{R}$$

$$\textbf{y} = [y_{1},y_{2},y_{3}]$$

$$y_{i} \in \mathbb{R}$$

We have that

* For any triangle $$T$$, $$\textrm{triangularity} (T) = 1$$.

In practice, calculating the triangularity exactly for a country is not possible. However, using optimisation methods, it is possible to produce a reasonable approximation.

# Method

I used the freely available countries dataset from [Natural Earth](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-countries/). The shapefile contains an ordered list of latitude and longitude points for the borders of 247 countries. Countries have one or more continuous parts (for example, each part might be an island belonging to the country). The part information is a series of indices that mark where the points for each part of the country begin and end. 

The first step in processing the data from the shapefiles was to convert the latitude and longitude coordinates using an azimuthal projection centered on each country. To do the conversion, I adapted some code from this [blog post](https://gciruelos.com/what-is-the-roundest-country.html). Then, using the python [Shapely](https://pypi.org/project/Shapely/) package, I converted the coordinates into a set of polygons for each country.

I wrote a function to calculate the similarity between the set of polygons that make up a country and a given triangle, making heavy use of the intersection and area methods from the Shapely package. I created an [R-Tree](https://en.wikipedia.org/wiki/R-tree) for each country, using the STRtree class in Shapely. This makes it fast to query which parts of the country might intersect with the triangle. Then, for each of these parts, I calculated the area of intersection with the triangle. These are summed and divided by the maximum of the total area of the country and the area of the triangle, as per the definition of similarity. This worked for all countries, apart from Antarctica, due to a problem with the polygon intersecting with itself.

To find the triangle that maximised the triangularity of each country, I began with an initial guess: three random points on the country's border. I then used the minimize function in scipy, using the Nelder-Mead simplex method, to find a triangle that maximised the triangularity of the country, i.e. minimising the cost function:

$$
f(\textbf{x},\textbf{y}) = 1 - \textrm{similarity} (C,T(\textbf{x},\textbf{y}))
$$

Interestingly, for a given country, the resulting triangles (and triangularities) after the optimisation varied quite significantly, depending on the initial guess. The problem is that the optimisation gets stuck in a local minimum of the cost function. In order to try and avoid local minima, I ran the optimisation routine 20 times for each country with a different initial guess each time, and used the estimate of triangularity from the best run. This is still no guarantee that local minima will be avoided, but does certainly help. 

The final step was simply to save an image of each country and its most similar triangle, and create a CSV file with estimated triangularity for each country.

# Results

The results shown here are all the countries in the Natural Earth dataset that are also countries according to [Encyclopedia Britannica](https://www.britannica.com/topic/list-of-countries-1993160). This means a slightly thinned down list of 196 countries from the original list of 247.

So these results would suggest that the most triangular country in the world is Nicaragua, followed closely by Bosnia and Herzegovina. (I will add that I found out an interesting fact about the flag of Bosnia and Herzegovina which is that it features a triangle that, according to [Wikipedia](https://en.wikipedia.org/wiki/Flag_of_Bosnia_and_Herzegovina), represents the approximate shape of the territory. Indeed, it does!)

I plotted the triangularity of each country by rank, labelling every 25th country. This shows that there is a quite gentle decrease in triangularity for the first 150 countries followed by a sharp drop off, seemingly due to countries that have multiple parts. The least triangular country is the Marshall Islands, hardly surprising considering the country is a collection of tiny islands spread over a large area of ocean. The least triangular country that isn't a group of islands is Vietnam. 

<img src="{{site.baseurl}}/assets/img/triangularity_graph.png"/>

Looking at pictures of the best triangles, it does seem like the optimisation has worked as intended, at least to the human eye. I think I would have picked similar triangles if I were doing the process manually.

Below is the final list of triangular countries. If you want to view the images in more detail, you can either open the images in a new tab or download them.

| Rank | Country | Triangularity | Image |
|-------|-------|-------|-------|
| 1 | Nicaragua | 0.918672 | <img src="{{site.baseurl}}/assets/img/Nicaragua.png" width="200"/> |
| 2 | Bosnia and Herzegovina | 0.913202 | <img src="{{site.baseurl}}/assets/img/Bosnia and Herzegovina.png" width="200"/> |
| 3 | Namibia | 0.907920 | <img src="{{site.baseurl}}/assets/img/Namibia.png" width="200"/> |
| 4 | Mauritania | 0.899915 | <img src="{{site.baseurl}}/assets/img/Mauritania.png" width="200"/> |
| 5 | Bolivia | 0.898253 | <img src="{{site.baseurl}}/assets/img/Bolivia.png" width="200"/> |
| 6 | Bhutan | 0.897231 | <img src="{{site.baseurl}}/assets/img/Bhutan.png" width="200"/> |
| 7 | Sri Lanka | 0.896619 | <img src="{{site.baseurl}}/assets/img/Sri Lanka.png" width="200"/> |
| 8 | Barbados | 0.892064 | <img src="{{site.baseurl}}/assets/img/Barbados.png" width="200"/> |
| 9 | Botswana | 0.889655 | <img src="{{site.baseurl}}/assets/img/Botswana.png" width="200"/> |
| 10 | Dominican Republic | 0.889356 | <img src="{{site.baseurl}}/assets/img/Dominican Republic.png" width="200"/> |
| 11 | Argentina | 0.888404 | <img src="{{site.baseurl}}/assets/img/Argentina.png" width="200"/> |
| 12 | Algeria | 0.887851 | <img src="{{site.baseurl}}/assets/img/Algeria.png" width="200"/> |
| 13 | Andorra | 0.887847 | <img src="{{site.baseurl}}/assets/img/Andorra.png" width="200"/> |
| 14 | Belarus | 0.886588 | <img src="{{site.baseurl}}/assets/img/Belarus.png" width="200"/> |
| 15 | Saint Lucia | 0.885393 | <img src="{{site.baseurl}}/assets/img/Saint Lucia.png" width="200"/> |
| 16 | Mongolia | 0.883835 | <img src="{{site.baseurl}}/assets/img/Mongolia.png" width="200"/> |
| 17 | Finland | 0.883464 | <img src="{{site.baseurl}}/assets/img/Finland.png" width="200"/> |
| 18 | East Timor | 0.883226 | <img src="{{site.baseurl}}/assets/img/East Timor.png" width="200"/> |
| 19 | Lithuania | 0.882420 | <img src="{{site.baseurl}}/assets/img/Lithuania.png" width="200"/> |
| 20 | Spain | 0.880991 | <img src="{{site.baseurl}}/assets/img/Spain.png" width="200"/> |
| 21 | Iraq | 0.880544 | <img src="{{site.baseurl}}/assets/img/Iraq.png" width="200"/> |
| 22 | Nigeria | 0.879989 | <img src="{{site.baseurl}}/assets/img/Nigeria.png" width="200"/> |
| 23 | Bahrain | 0.879084 | <img src="{{site.baseurl}}/assets/img/Bahrain.png" width="200"/> |
| 24 | Central African Republic | 0.878863 | <img src="{{site.baseurl}}/assets/img/Central African Republic.png" width="200"/> |
| 25 | Niger | 0.878834 | <img src="{{site.baseurl}}/assets/img/Niger.png" width="200"/> |
| 26 | Cambodia | 0.878727 | <img src="{{site.baseurl}}/assets/img/Cambodia.png" width="200"/> |
| 27 | Libya | 0.878326 | <img src="{{site.baseurl}}/assets/img/Libya.png" width="200"/> |
| 28 | Lebanon | 0.877776 | <img src="{{site.baseurl}}/assets/img/Lebanon.png" width="200"/> |
| 29 | Chad | 0.876590 | <img src="{{site.baseurl}}/assets/img/Chad.png" width="200"/> |
| 30 | Dominica | 0.875869 | <img src="{{site.baseurl}}/assets/img/Dominica.png" width="200"/> |
| 31 | Taiwan | 0.875187 | <img src="{{site.baseurl}}/assets/img/Taiwan.png" width="200"/> |
| 32 | Brazil | 0.874508 | <img src="{{site.baseurl}}/assets/img/Brazil.png" width="200"/> |
| 33 | Saudi Arabia | 0.874421 | <img src="{{site.baseurl}}/assets/img/Saudi Arabia.png" width="200"/> |
| 34 | Zimbabwe | 0.874192 | <img src="{{site.baseurl}}/assets/img/Zimbabwe.png" width="200"/> |
| 35 | Ecuador | 0.872448 | <img src="{{site.baseurl}}/assets/img/Ecuador.png" width="200"/> |
| 36 | Venezuela | 0.871525 | <img src="{{site.baseurl}}/assets/img/Venezuela.png" width="200"/> |
| 37 | Uruguay | 0.869709 | <img src="{{site.baseurl}}/assets/img/Uruguay.png" width="200"/> |
| 38 | Benin | 0.868978 | <img src="{{site.baseurl}}/assets/img/Benin.png" width="200"/> |
| 39 | Liberia | 0.868948 | <img src="{{site.baseurl}}/assets/img/Liberia.png" width="200"/> |
| 40 | Madagascar | 0.867805 | <img src="{{site.baseurl}}/assets/img/Madagascar.png" width="200"/> |
| 41 | Democratic Republic of the Congo | 0.866893 | <img src="{{site.baseurl}}/assets/img/Democratic Republic of the Congo.png" width="200"/> |
| 42 | Ethiopia | 0.866403 | <img src="{{site.baseurl}}/assets/img/Ethiopia.png" width="200"/> |
| 43 | Syria | 0.865422 | <img src="{{site.baseurl}}/assets/img/Syria.png" width="200"/> |
| 44 | Slovenia | 0.864037 | <img src="{{site.baseurl}}/assets/img/Slovenia.png" width="200"/> |
| 45 | Liechtenstein | 0.863911 | <img src="{{site.baseurl}}/assets/img/Liechtenstein.png" width="200"/> |
| 46 | Afghanistan | 0.863858 | <img src="{{site.baseurl}}/assets/img/Afghanistan.png" width="200"/> |
| 47 | Honduras | 0.863776 | <img src="{{site.baseurl}}/assets/img/Honduras.png" width="200"/> |
| 48 | Burkina Faso | 0.863649 | <img src="{{site.baseurl}}/assets/img/Burkina Faso.png" width="200"/> |
| 49 | San Marino | 0.863048 | <img src="{{site.baseurl}}/assets/img/San Marino.png" width="200"/> |
| 50 | South Africa | 0.860611 | <img src="{{site.baseurl}}/assets/img/South Africa.png" width="200"/> |
| 51 | Vatican | 0.860316 | <img src="{{site.baseurl}}/assets/img/Vatican.png" width="200"/> |
| 52 | Belgium | 0.859731 | <img src="{{site.baseurl}}/assets/img/Belgium.png" width="200"/> |
| 53 | Poland | 0.859164 | <img src="{{site.baseurl}}/assets/img/Poland.png" width="200"/> |
| 54 | Georgia | 0.859137 | <img src="{{site.baseurl}}/assets/img/Georgia.png" width="200"/> |
| 55 | Austria | 0.858090 | <img src="{{site.baseurl}}/assets/img/Austria.png" width="200"/> |
| 56 | Hungary | 0.857817 | <img src="{{site.baseurl}}/assets/img/Hungary.png" width="200"/> |
| 57 | Ghana | 0.857068 | <img src="{{site.baseurl}}/assets/img/Ghana.png" width="200"/> |
| 58 | Nepal | 0.857044 | <img src="{{site.baseurl}}/assets/img/Nepal.png" width="200"/> |
| 59 | Slovakia | 0.856166 | <img src="{{site.baseurl}}/assets/img/Slovakia.png" width="200"/> |
| 60 | El Salvador | 0.856155 | <img src="{{site.baseurl}}/assets/img/El Salvador.png" width="200"/> |
| 61 | Republic of Serbia | 0.856100 | <img src="{{site.baseurl}}/assets/img/Republic of Serbia.png" width="200"/> |
| 62 | Armenia | 0.856031 | <img src="{{site.baseurl}}/assets/img/Armenia.png" width="200"/> |
| 63 | Morocco | 0.855741 | <img src="{{site.baseurl}}/assets/img/Morocco.png" width="200"/> |
| 64 | North Korea | 0.855623 | <img src="{{site.baseurl}}/assets/img/North Korea.png" width="200"/> |
| 65 | South Sudan | 0.854143 | <img src="{{site.baseurl}}/assets/img/South Sudan.png" width="200"/> |
| 66 | Estonia | 0.853185 | <img src="{{site.baseurl}}/assets/img/Estonia.png" width="200"/> |
| 67 | Yemen | 0.852291 | <img src="{{site.baseurl}}/assets/img/Yemen.png" width="200"/> |
| 68 | Mauritius | 0.852287 | <img src="{{site.baseurl}}/assets/img/Mauritius.png" width="200"/> |
| 69 | Burundi | 0.852197 | <img src="{{site.baseurl}}/assets/img/Burundi.png" width="200"/> |
| 70 | Cameroon | 0.852041 | <img src="{{site.baseurl}}/assets/img/Cameroon.png" width="200"/> |
| 71 | Kuwait | 0.852006 | <img src="{{site.baseurl}}/assets/img/Kuwait.png" width="200"/> |
| 72 | Luxembourg | 0.851825 | <img src="{{site.baseurl}}/assets/img/Luxembourg.png" width="200"/> |
| 73 | United Republic of Tanzania | 0.850960 | <img src="{{site.baseurl}}/assets/img/United Republic of Tanzania.png" width="200"/> |
| 74 | Monaco | 0.850811 | <img src="{{site.baseurl}}/assets/img/Monaco.png" width="200"/> |
| 75 | Kyrgyzstan | 0.849767 | <img src="{{site.baseurl}}/assets/img/Kyrgyzstan.png" width="200"/> |
| 76 | Ivory Coast | 0.849450 | <img src="{{site.baseurl}}/assets/img/Ivory Coast.png" width="200"/> |
| 77 | Guinea | 0.849425 | <img src="{{site.baseurl}}/assets/img/Guinea.png" width="200"/> |
| 78 | Romania | 0.849085 | <img src="{{site.baseurl}}/assets/img/Romania.png" width="200"/> |
| 79 | South Korea | 0.847823 | <img src="{{site.baseurl}}/assets/img/South Korea.png" width="200"/> |
| 80 | Uganda | 0.847599 | <img src="{{site.baseurl}}/assets/img/Uganda.png" width="200"/> |
| 81 | Singapore | 0.847415 | <img src="{{site.baseurl}}/assets/img/Singapore.png" width="200"/> |
| 82 | Kenya | 0.847016 | <img src="{{site.baseurl}}/assets/img/Kenya.png" width="200"/> |
| 83 | Australia | 0.844598 | <img src="{{site.baseurl}}/assets/img/Australia.png" width="200"/> |
| 84 | Cyprus | 0.844447 | <img src="{{site.baseurl}}/assets/img/Cyprus.png" width="200"/> |
| 85 | United Arab Emirates | 0.844364 | <img src="{{site.baseurl}}/assets/img/United Arab Emirates.png" width="200"/> |
| 86 | Montenegro | 0.842538 | <img src="{{site.baseurl}}/assets/img/Montenegro.png" width="200"/> |
| 87 | Macedonia | 0.842095 | <img src="{{site.baseurl}}/assets/img/Macedonia.png" width="200"/> |
| 88 | Ireland | 0.841491 | <img src="{{site.baseurl}}/assets/img/Ireland.png" width="200"/> |
| 89 | Jamaica | 0.840451 | <img src="{{site.baseurl}}/assets/img/Jamaica.png" width="200"/> |
| 90 | Senegal | 0.840054 | <img src="{{site.baseurl}}/assets/img/Senegal.png" width="200"/> |
| 91 | Suriname | 0.839448 | <img src="{{site.baseurl}}/assets/img/Suriname.png" width="200"/> |
| 92 | Turkey | 0.838539 | <img src="{{site.baseurl}}/assets/img/Turkey.png" width="200"/> |
| 93 | Pakistan | 0.838210 | <img src="{{site.baseurl}}/assets/img/Pakistan.png" width="200"/> |
| 94 | Nauru | 0.838165 | <img src="{{site.baseurl}}/assets/img/Nauru.png" width="200"/> |
| 95 | Iceland | 0.837977 | <img src="{{site.baseurl}}/assets/img/Iceland.png" width="200"/> |
| 96 | Qatar | 0.837840 | <img src="{{site.baseurl}}/assets/img/Qatar.png" width="200"/> |
| 97 | Mali | 0.837820 | <img src="{{site.baseurl}}/assets/img/Mali.png" width="200"/> |
| 98 | Sudan | 0.837108 | <img src="{{site.baseurl}}/assets/img/Sudan.png" width="200"/> |
| 99 | Albania | 0.836750 | <img src="{{site.baseurl}}/assets/img/Albania.png" width="200"/> |
| 100 | Czechia | 0.836644 | <img src="{{site.baseurl}}/assets/img/Czechia.png" width="200"/> |
| 101 | Togo | 0.836207 | <img src="{{site.baseurl}}/assets/img/Togo.png" width="200"/> |
| 102 | Iran | 0.835827 | <img src="{{site.baseurl}}/assets/img/Iran.png" width="200"/> |
| 103 | Sierra Leone | 0.835686 | <img src="{{site.baseurl}}/assets/img/Sierra Leone.png" width="200"/> |
| 104 | Kosovo | 0.834849 | <img src="{{site.baseurl}}/assets/img/Kosovo.png" width="200"/> |
| 105 | eSwatini | 0.834460 | <img src="{{site.baseurl}}/assets/img/eSwatini.png" width="200"/> |
| 106 | Gabon | 0.834010 | <img src="{{site.baseurl}}/assets/img/Gabon.png" width="200"/> |
| 107 | Bulgaria | 0.833776 | <img src="{{site.baseurl}}/assets/img/Bulgaria.png" width="200"/> |
| 108 | Angola | 0.832625 | <img src="{{site.baseurl}}/assets/img/Angola.png" width="200"/> |
| 109 | Oman | 0.832474 | <img src="{{site.baseurl}}/assets/img/Oman.png" width="200"/> |
| 110 | Lesotho | 0.832316 | <img src="{{site.baseurl}}/assets/img/Lesotho.png" width="200"/> |
| 111 | Germany | 0.832018 | <img src="{{site.baseurl}}/assets/img/Germany.png" width="200"/> |
| 112 | Rwanda | 0.831329 | <img src="{{site.baseurl}}/assets/img/Rwanda.png" width="200"/> |
| 113 | Switzerland | 0.830879 | <img src="{{site.baseurl}}/assets/img/Switzerland.png" width="200"/> |
| 114 | Belize | 0.830612 | <img src="{{site.baseurl}}/assets/img/Belize.png" width="200"/> |
| 115 | Guatemala | 0.830442 | <img src="{{site.baseurl}}/assets/img/Guatemala.png" width="200"/> |
| 116 | Turkmenistan | 0.830046 | <img src="{{site.baseurl}}/assets/img/Turkmenistan.png" width="200"/> |
| 117 | Peru | 0.829379 | <img src="{{site.baseurl}}/assets/img/Peru.png" width="200"/> |
| 118 | Moldova | 0.829254 | <img src="{{site.baseurl}}/assets/img/Moldova.png" width="200"/> |
| 119 | Myanmar | 0.827051 | <img src="{{site.baseurl}}/assets/img/Myanmar.png" width="200"/> |
| 120 | China | 0.825111 | <img src="{{site.baseurl}}/assets/img/China.png" width="200"/> |
| 121 | Kazakhstan | 0.825079 | <img src="{{site.baseurl}}/assets/img/Kazakhstan.png" width="200"/> |
| 122 | Saint Vincent and the Grenadines | 0.824970 | <img src="{{site.baseurl}}/assets/img/Saint Vincent and the Grenadines.png" width="200"/> |
| 123 | Latvia | 0.824709 | <img src="{{site.baseurl}}/assets/img/Latvia.png" width="200"/> |
| 124 | Djibouti | 0.824624 | <img src="{{site.baseurl}}/assets/img/Djibouti.png" width="200"/> |
| 125 | Equatorial Guinea | 0.819325 | <img src="{{site.baseurl}}/assets/img/Equatorial Guinea.png" width="200"/> |
| 126 | Jordan | 0.818978 | <img src="{{site.baseurl}}/assets/img/Jordan.png" width="200"/> |
| 127 | Netherlands | 0.818781 | <img src="{{site.baseurl}}/assets/img/Netherlands.png" width="200"/> |
| 128 | Paraguay | 0.816157 | <img src="{{site.baseurl}}/assets/img/Paraguay.png" width="200"/> |
| 129 | Guinea-Bissau | 0.815927 | <img src="{{site.baseurl}}/assets/img/Guinea-Bissau.png" width="200"/> |
| 130 | India | 0.814979 | <img src="{{site.baseurl}}/assets/img/India.png" width="200"/> |
| 131 | Costa Rica | 0.814253 | <img src="{{site.baseurl}}/assets/img/Costa Rica.png" width="200"/> |
| 132 | Ukraine | 0.813770 | <img src="{{site.baseurl}}/assets/img/Ukraine.png" width="200"/> |
| 133 | Colombia | 0.813659 | <img src="{{site.baseurl}}/assets/img/Colombia.png" width="200"/> |
| 134 | Grenada | 0.812532 | <img src="{{site.baseurl}}/assets/img/Grenada.png" width="200"/> |
| 135 | Egypt | 0.812297 | <img src="{{site.baseurl}}/assets/img/Egypt.png" width="200"/> |
| 136 | Portugal | 0.811106 | <img src="{{site.baseurl}}/assets/img/Portugal.png" width="200"/> |
| 137 | Republic of the Congo | 0.811063 | <img src="{{site.baseurl}}/assets/img/Republic of the Congo.png" width="200"/> |
| 138 | Tunisia | 0.807452 | <img src="{{site.baseurl}}/assets/img/Tunisia.png" width="200"/> |
| 139 | Trinidad and Tobago | 0.804956 | <img src="{{site.baseurl}}/assets/img/Trinidad and Tobago.png" width="200"/> |
| 140 | Sweden | 0.804381 | <img src="{{site.baseurl}}/assets/img/Sweden.png" width="200"/> |
| 141 | Somalia | 0.802796 | <img src="{{site.baseurl}}/assets/img/Somalia.png" width="200"/> |
| 142 | Malawi | 0.802539 | <img src="{{site.baseurl}}/assets/img/Malawi.png" width="200"/> |
| 143 | Russia | 0.800247 | <img src="{{site.baseurl}}/assets/img/Russia.png" width="200"/> |
| 144 | Zambia | 0.799161 | <img src="{{site.baseurl}}/assets/img/Zambia.png" width="200"/> |
| 145 | Guyana | 0.798601 | <img src="{{site.baseurl}}/assets/img/Guyana.png" width="200"/> |
| 146 | Azerbaijan | 0.796271 | <img src="{{site.baseurl}}/assets/img/Azerbaijan.png" width="200"/> |
| 147 | Uzbekistan | 0.795089 | <img src="{{site.baseurl}}/assets/img/Uzbekistan.png" width="200"/> |
| 148 | Thailand | 0.794683 | <img src="{{site.baseurl}}/assets/img/Thailand.png" width="200"/> |
| 149 | São Tomé and Principe | 0.793244 | <img src="{{site.baseurl}}/assets/img/São Tomé and Principe.png" width="200"/> |
| 150 | Bangladesh | 0.791780 | <img src="{{site.baseurl}}/assets/img/Bangladesh.png" width="200"/> |
| 151 | United Kingdom | 0.791691 | <img src="{{site.baseurl}}/assets/img/United Kingdom.png" width="200"/> |
| 152 | Eritrea | 0.789674 | <img src="{{site.baseurl}}/assets/img/Eritrea.png" width="200"/> |
| 153 | Malta | 0.786289 | <img src="{{site.baseurl}}/assets/img/Malta.png" width="200"/> |
| 154 | Mozambique | 0.786196 | <img src="{{site.baseurl}}/assets/img/Mozambique.png" width="200"/> |
| 155 | Tajikistan | 0.785365 | <img src="{{site.baseurl}}/assets/img/Tajikistan.png" width="200"/> |
| 156 | Laos | 0.781655 | <img src="{{site.baseurl}}/assets/img/Laos.png" width="200"/> |
| 157 | United States of America | 0.771845 | <img src="{{site.baseurl}}/assets/img/United States of America.png" width="200"/> |
| 158 | Mexico | 0.762001 | <img src="{{site.baseurl}}/assets/img/Mexico.png" width="200"/> |
| 159 | Israel | 0.758537 | <img src="{{site.baseurl}}/assets/img/Israel.png" width="200"/> |
| 160 | Brunei | 0.752157 | <img src="{{site.baseurl}}/assets/img/Brunei.png" width="200"/> |
| 161 | Papua New Guinea | 0.748362 | <img src="{{site.baseurl}}/assets/img/Papua New Guinea.png" width="200"/> |
| 162 | Canada | 0.747720 | <img src="{{site.baseurl}}/assets/img/Canada.png" width="200"/> |
| 163 | Cuba | 0.735471 | <img src="{{site.baseurl}}/assets/img/Cuba.png" width="200"/> |
| 164 | France | 0.734136 | <img src="{{site.baseurl}}/assets/img/France.png" width="200"/> |
| 165 | Italy | 0.729978 | <img src="{{site.baseurl}}/assets/img/Italy.png" width="200"/> |
| 166 | New Zealand | 0.723253 | <img src="{{site.baseurl}}/assets/img/New Zealand.png" width="200"/> |
| 167 | Samoa | 0.719025 | <img src="{{site.baseurl}}/assets/img/Samoa.png" width="200"/> |
| 168 | Greece | 0.710513 | <img src="{{site.baseurl}}/assets/img/Greece.png" width="200"/> |
| 169 | Haiti | 0.709348 | <img src="{{site.baseurl}}/assets/img/Haiti.png" width="200"/> |
| 170 | Gambia | 0.706181 | <img src="{{site.baseurl}}/assets/img/Gambia.png" width="200"/> |
| 171 | Denmark | 0.702239 | <img src="{{site.baseurl}}/assets/img/Denmark.png" width="200"/> |
| 172 | Chile | 0.694678 | <img src="{{site.baseurl}}/assets/img/Chile.png" width="200"/> |
| 173 | Saint Kitts and Nevis | 0.694223 | <img src="{{site.baseurl}}/assets/img/Saint Kitts and Nevis.png" width="200"/> |
| 174 | Palau | 0.685319 | <img src="{{site.baseurl}}/assets/img/Palau.png" width="200"/> |
| 175 | Croatia | 0.667609 | <img src="{{site.baseurl}}/assets/img/Croatia.png" width="200"/> |
| 176 | Panama | 0.650465 | <img src="{{site.baseurl}}/assets/img/Panama.png" width="200"/> |
| 177 | Japan | 0.649205 | <img src="{{site.baseurl}}/assets/img/Japan.png" width="200"/> |
| 178 | Fiji | 0.645017 | <img src="{{site.baseurl}}/assets/img/Fiji.png" width="200"/> |
| 179 | Norway | 0.635156 | <img src="{{site.baseurl}}/assets/img/Norway.png" width="200"/> |
| 180 | Vietnam | 0.629038 | <img src="{{site.baseurl}}/assets/img/Vietnam.png" width="200"/> |
| 181 | Antigua and Barbuda | 0.614450 | <img src="{{site.baseurl}}/assets/img/Antigua and Barbuda.png" width="200"/> |
| 182 | Comoros | 0.612388 | <img src="{{site.baseurl}}/assets/img/Comoros.png" width="200"/> |
| 183 | Malaysia | 0.561826 | <img src="{{site.baseurl}}/assets/img/Malaysia.png" width="200"/> |
| 184 | Federated States of Micronesia | 0.536417 | <img src="{{site.baseurl}}/assets/img/Federated States of Micronesia.png" width="200"/> |
| 185 | Philippines | 0.536229 | <img src="{{site.baseurl}}/assets/img/Philippines.png" width="200"/> |
| 186 | Indonesia | 0.493143 | <img src="{{site.baseurl}}/assets/img/Indonesia.png" width="200"/> |
| 187 | Vanuatu | 0.473048 | <img src="{{site.baseurl}}/assets/img/Vanuatu.png" width="200"/> |
| 188 | Tonga | 0.429500 | <img src="{{site.baseurl}}/assets/img/Tonga.png" width="200"/> |
| 189 | The Bahamas | 0.405978 | <img src="{{site.baseurl}}/assets/img/The Bahamas.png" width="200"/> |
| 190 | Seychelles | 0.388652 | <img src="{{site.baseurl}}/assets/img/Seychelles.png" width="200"/> |
| 191 | Solomon Islands | 0.368949 | <img src="{{site.baseurl}}/assets/img/Solomon Islands.png" width="200"/> |
| 192 | Cabo Verde | 0.309594 | <img src="{{site.baseurl}}/assets/img/Cabo Verde.png" width="200"/> |
| 193 | Maldives | 0.203665 | <img src="{{site.baseurl}}/assets/img/Maldives.png" width="200"/> |
| 194 | Tuvalu | 0.144069 | <img src="{{site.baseurl}}/assets/img/Tuvalu.png" width="200"/> |
| 195 | Kiribati | 0.138684 | <img src="{{site.baseurl}}/assets/img/Kiribati.png" width="200"/> |
| 196 | Marshall Islands | 0.084072 | <img src="{{site.baseurl}}/assets/img/Marshall Islands.png" width="200"/> |

# Code
Python code to calculate the triangularity of every country in Natural Earth's ne_10m_admin_0_countries.shp shapefile.

{% highlight python %}
import shapefile
import pyproj
import random

import numpy as np
import matplotlib.pyplot as plt
import pandas as pd

from shapely.geometry import Polygon
from shapely.strtree import STRtree
from scipy.optimize import minimize

def transform_points(points):
    """
    converts from lat lon to azimuthal projection on center of points
    - returns converted points
    (adapted from https://gciruelos.com/what-is-the-roundest-country.html)
    """

    xs = [x for x,_ in points]
    ys = [y for _,y in points]
    lon = np.mean(xs)
    lat = np.mean(ys)
    FROM_PROJ = '+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs'
    to_proj = '+proj=aeqd +R=6371000 +lat_0=%.2f +lon_0=%.2f ' \
                      '+no_defs' % (lat, lon)
    tpoints = [pyproj.transform(pyproj.Proj(FROM_PROJ),pyproj.Proj(to_proj),x,y)
                                for x,y in points]
    return tpoints

def get_polys(points, parts):
    """
    - returns polygons from points and parts
    """

    polys = []

    for j,k in enumerate(parts):

        if j == len(parts)-1:
            poly = Polygon(points[k:])
        else:
            k_add1 = parts[j+1]
            poly = Polygon(points[k:k_add1])

        polys.append(poly)
        
    return polys

def similarity(triangle, polys):
    """
    - returns similarity for given triangle (polygon) and polygons (country)
    """

    # create an STR tree from polygons and find which polygons intersect the triangle
    tree_polys = STRtree(polys)
    intersecting_polys = tree_polys.query(triangle)

    country_area = sum([poly.area for poly in polys])
    triangle_area = triangle.area
    intersection_area = sum([poly.intersection(triangle).area for poly in intersecting_polys])

    triangularity = intersection_area / max(triangle_area, country_area)
    
    return triangularity

def run_simplex(x0, polys):
    """
    maximise triangularity for a set of polygons (country) using Nelder-Mead method
    given initial guess x0
    where first 3 elements of x0 are coordinates [x1,x2,x3] and last three elements are [y1,y2,y3]
    - returns an OptimizeResult object (see https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.OptimizeResult.html#scipy.optimize.OptimizeResult)
    """

    def cost_function(x):
        triangle = Polygon((x[i],x[i+3]) for i in range(3))
        cost = 1 - similarity(triangle, polys)
        return cost

    # minimize cost function (1 - triangularity) to find best triangle
    res = minimize(cost_function, x0, method = "Nelder-Mead")
    
    return res

def run_trials(n, polys, points):
    """
    runs optimisation routine n times with randomised initial guesses (random three points on country's border)
    - returns best triangularity found, and corresponging triangle (polygon)
    """

    best_triangularity = 0
    
    # run with n times with different initial guesses
    for i in range(n):
        
        # the initial guess first 3 are x-coords, last 3 are y-coords
        rand_points = random.sample(set(points), 3)
        x0 = [point[0] for point in rand_points] + [point[1] for point in rand_points]
        
        res = run_simplex(x0, polys)
        triangularity = 1 - res["fun"]
        print(triangularity)
        x = res["x"]
        
        if triangularity >= best_triangularity:
            best_triangularity = triangularity
            best_x = x
    
    triangle = Polygon((best_x[i],best_x[i+3]) for i in range(3))
    
    return best_triangularity, triangle

def main():
    """
    main function runs through all countries in shape file
    finds optimal triangularity after n runs of optimisation routine per country
    - saves corresponding trianglularity per country in a CSV file 
    - saves plot of each country with optimal triangle
    """

    # PARAMETERS
    # input shape file data
    input_data = "country//ne_10m_admin_0_countries.shp" 
    # number of times optimisation routine is run per country (with different initial guess)
    n = 20                                    

    # read shape file and get shapes
    sf = shapefile.Reader(input_data)
    shapes = sf.shapes()

    # list of countries
    countries = [sf.record(i)[8] for i in range(len(shapes))]
    num_countries = len(countries)

    opt_triangularities = []

    for i, country in enumerate(countries):
        try:
            print("(" + str(i+1) + "/" + str(num_countries) + ") " + country)

            s = shapes[i]

            # get (transformed points) and parts of country
            tpoints = transform_points(s.points)
            parts = s.parts

            # get polygons of country
            polys = get_polys(tpoints, parts)

            # optimisation
            xs = [x for x,_ in tpoints]
            ys = [y for _,y in tpoints]
            min_xs = min(xs)
            max_xs = max(xs)
            min_ys = min(ys)
            max_ys = max(ys)

            print("Optimising...")
            triangularity, triangle = run_trials(n, polys, tpoints)

            opt_triangularities.append(triangularity)

            print("Triangularity: " + str(triangularity))

            # set up figure for plotting
            fig, ax = plt.subplots(figsize = (8,8))

            ys_range = max_ys - min_ys
            xs_range = max_xs - min_xs
            plot_range = max(xs_range, ys_range)
            xs_center = np.mean((min_xs, max_xs))
            ys_center = np.mean((min_ys, max_ys))
            ax.set_xlim(xs_center-0.75*plot_range, xs_center+0.75*plot_range)
            ax.set_ylim(ys_center-0.75*plot_range, ys_center+0.75*plot_range)

            ax.spines['top'].set_visible(False)
            ax.spines['right'].set_visible(False)
            ax.spines['bottom'].set_visible(False)
            ax.spines['left'].set_visible(False)
            ax.get_xaxis().set_ticks([])
            ax.get_yaxis().set_ticks([])

            # plot the country
            for poly in polys:
                plt.plot(*poly.exterior.xy, color="black", linewidth=1)

            # plot triangle
            plt.plot(*triangle.exterior.xy, color="red", linewidth=1)

            plt.savefig(country + ".png", dpi=400)
            plt.close('all')
            print("\n")

        except: 
            opt_triangularities.append(np.nan)
            print("\n")

    # make dataframe and save results
    df_result = pd.DataFrame({"Country": countries,
                            "Triangularity": opt_triangularities})
    df_result = df_result.sort_values(["Triangularity"], ascending = False)
    df_result["Rank"] = np.arange(1,num_countries+1)
    df_result = df_result.set_index("Rank")
    df_result.to_csv("result.csv")

    print("Results Saved")
    print("Finished")

if __name__ == "__main__":
    main()
{% endhighlight %}
---
layout: page
# menu: hidden
# menuservices: services
order: 6
title: Property Regulations Map
permalink: /property-regulations-map/

image_sliders: [small_slider]
image_slider_selector: "small_slider"

contains_map: true
---
<div class="map-content" style="display:flex; align-items:center; flex-direction:column;">
<div class="input-group">
	<input type="text" class="form-control input-field" id="geocodeField" placeholder="Enter civic or intersection address"/>
									<button id="geocodeBtn" class="btn btn-default" type="button" title="Search">
										<span class="glyphicon glyphicon-search" aria-label="Search"></span>
									</button>
</div>

<div id="map" style="width: 90%; height: 400px;"></div>
</div>
<script src="{{ site.url }}{{ site.baseurl }}maps/regulatory_files/regulatory_map.js"></script>

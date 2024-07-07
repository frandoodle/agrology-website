---
layout: page
# menu: hidden
# menuservices: services
order: 6
title: Property Regulations Map2
permalink: /property-regulations-map2/

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

<div id="map" style="width: 140%; height: 400px;"></div>
</div>
<script>

	// prevent an routing from being done until everything is set up
		var settingUp = true;

		L.NumberedIcon = L.Icon.extend({
			options: {
    			number: '',
				iconUrl: 'img/marker-icon-hole.png',
    			iconSize: new L.Point(25, 41),
				iconAnchor: new L.Point(13, 41),
				popupAnchor: new L.Point(0, -33),
				shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
				shadowSize: [41, 41]
			},

			createIcon: function () {
				var div = document.createElement('div');
				var img = this._createImg(this.options['iconUrl']);
				var numdiv = document.createElement('div');
				numdiv.setAttribute ( "class", "number" );
				numdiv.innerHTML = this.options['number'] || '';
				div.appendChild ( img );
				div.appendChild ( numdiv );
				this._setIconStyles(div, 'icon');
				return div;
			},

		});

		// parse query params
		var queryParams = new URL(document.location).searchParams;

		// Keys
		var BING_KEY = 'ArFQcMQQdw-tWyNSzt5Mafl-kq4I6naIWvX7YZjq-P20-f-txYOcMzPnl0yoNvvn';
		var DATABC_APIKEY = "11dd756f680c47b5aef5093d95543738";

		// API URLs
		var gcApi = "https://geocoder.api.gov.bc.ca/";
		var routeApi = "https://router.api.gov.bc.ca/";
		var routeMethod = 'POST';
		//var routeMethod = 'GET';
		switch(queryParams.get('gc')) {
			case 'tst':
				gcApi = "https://geocodertst.api.gov.bc.ca/";
				break;
			case 'dlv':
			  gcApi = "https://geocoderdlv.api.gov.bc.ca/";
				break;
			case 'rri':
				gcApi = "https://ssl.refractions.net/ols/geocoder/";
				break
			case 'local':
				gcApi = "http://localhost:8080/";
				break;
			case 'dip':
				gcApi = "https://data-integration-geocoder-prod.apps.silver.devops.gov.bc.ca/";
		}

		switch(queryParams.get('rt')) {
			case 'tst':
				routeApi = "https://routertst.api.gov.bc.ca/";
				break;
			case 'dlv':
				routeApi = "https://routerdlv.api.gov.bc.ca/";
				break;
		 	case 'rri':
				routeApi = "https://ssl.refractions.net/ols/router/";
				break;
			case 'local':
				routeApi = "http://localhost:8080/";
				break;
			case 'dip':
				routeApi = "https://data-integration-router-api-prod.apps.silver.devops.gov.bc.ca/";
				break;
			default:
				routeMethod = 'GET';
		}

		var routeHeaders = {apiKey: DATABC_APIKEY};
		var routeQuery = '';
		if(routeMethod == 'GET') {
			routeHeaders = {};
			routeQuery = 'apikey=' + DATABC_APIKEY
		}

		var xmdx = 5000;
		if(queryParams.get('xmdx')) {
			var num = parseInt(queryParams.get('xmdx'));
			if(!isNaN(num)) {
				xmdx = num;
			}
		}

		var matchPrecisionNot = "";
		if('noStreets' in queryParams) {
			matchPrecisionNot = "street";
		}

		var defaultLocationDescriptor = 'parcelPoint';
		if(queryParams.get('locationDescriptor') == 'accessPoint'
				|| queryParams.get('locationDescriptor') == 'routingPoint') {
			defaultLocationDescriptor = queryParams.get('locationDescriptor');
		}

	const map = L.map('map', {
			minZoom: 4,
			maxZoom: 20,
			zoomControl: true
		}).setView([48.44, -123.43], 12);

	L.control.locate({
			position: 'bottomright',
			icon: 'glyphicon glyphicon-map-marker',
			iconLoading: 'glyphicon glyphicon-time',
			locateOptions: {
				maxZoom: 16
			}
		}).addTo(map);

	var baseLayers = {};
	var overlays = {};

	var baseErrorMsg = "An unexpected server error occurred; No response from server.\n";

	const osmLayer = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
		minZoom: 4,
		maxZoom: 20,
		zoomControl: false,
		attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
	}).addTo(map);

	baseLayers['OpenStreetMap'] = osmLayer;

function makePopupText(props) {
			var str = '<span class="popup-title">' + props.fullAddress + '</span> ';
			return str;
		}

// reverse geocode on map Click
		map.on('click', function(e) {
			if($('#tabs-2').hasClass('active')) {
				$('#addAddrField').val(e.latlng.lat.toFixed(5) + "," + e.latlng.lng.toFixed(5));
			}
		});

	// reusable function for suggesting geocode autocompletion options
		function geocodeSuggest(request, response, options) {
			var params = {
				minScore: 50,
				maxResults: 5,
				echo: true,
				brief: true,
				autoComplete: true,
				//exactSpelling: $('#exactSpellingChk').is(':checked'),
				matchPrecisionNot: matchPrecisionNot,
				locationDescriptor: defaultLocationDescriptor,
				addressString: request.term
			};
			$.extend(params, options);
			$.ajax({
				url: gcApi + "addresses.json",
				data: params,
				success: function(data) {
					var list = [];
					if(data.features && data.features.length > 0) {
						list = data.features.map( function(item) {
							return {
								value: item.properties.fullAddress,
								data: item
							}
						});
					}
					response(list);
				},
				error: function() {
					response([]);
				}
			});
		}

		// function to convert json or a latlng into a layer and put it on the map
		function receiveGeocode(data, center=true) {
			var layer;
			if(data instanceof L.LatLng) {
				layer = latLngToLayer(data);
			} else {
				layer = L.geoJson(data, {
					onEachFeature: function(feature, layer) {
						layer.bindPopup(makePopupText(feature.properties));
						layer.options['title'] = feature.properties.fullAddress;
						//layer.bindTooltip(feature.properties.fullAddress, {
						//		permanent: true
						//	}).openTooltip();
					}
				});
				var feature = data;
				if(data.features) {
					feature = data.features[0];
				}
				lookupAdminAreas(feature.properties.fullAddress, feature.geometry.coordinates);
			}
			if(geocodeLayer) {
				map.removeLayer(geocodeLayer);
			}
			geocodeLayer = layer;
			layer.addTo(map);
			centerMap(geocodeLayer.getBounds(), center);
		}

		function lookupAdminAreas(address, coords) {
			$('#adminAreaInfo').html("<b>" + address + "</b><br/>is in the following admin areas:<br/>");
			lookupCHSAInfo(coords);
		}

		function lookupCHSAInfo(coords) {
			var params = {
				service: "WFS",
				version: "1.0.0",
				request: "GetFeature",
				typeName: "pub:WHSE_ADMIN_BOUNDARIES.BCHA_CMNTY_HEALTH_SERV_AREA_SP",
				srsname: "EPSG:4326",
				cql_filter: "INTERSECTS(SHAPE,SRID=4326;POINT(" + coords[0] + " " + coords[1] + "))",
				propertyName: "CMNTY_HLTH_SERV_AREA_CODE,CMNTY_HLTH_SERV_AREA_NAME,LOCAL_HLTH_AREA_CODE,LOCAL_HLTH_AREA_NAME,HLTH_SERVICE_DLVR_AREA_CODE,HLTH_SERVICE_DLVR_AREA_NAME,HLTH_AUTHORITY_CODE,HLTH_AUTHORITY_NAME",
				outputFormat: "application/json",
			};

			$.ajax({
				url: "https://openmaps.gov.bc.ca/geo/pub/ows",
				data: params,
				type: "GET",
				dataType: "json",
				success: function (response, textStatus, jqXHR) {
					var f = response.features[0];
					$('#adminAreaInfo').append("CHSA: " + f.properties['CMNTY_HLTH_SERV_AREA_NAME'] + " (" + f.properties['CMNTY_HLTH_SERV_AREA_CODE'] + ")<br/>");
					$('#adminAreaInfo').append("LHA: " + f.properties['LOCAL_HLTH_AREA_NAME'] + " (" + f.properties['LOCAL_HLTH_AREA_CODE'] + ")<br/>");
					$('#adminAreaInfo').append("CSDA: " + f.properties['HLTH_SERVICE_DLVR_AREA_NAME'] + " (" + f.properties['HLTH_SERVICE_DLVR_AREA_CODE'] + ")<br/>");
					$('#adminAreaInfo').append("HA: " + f.properties['HLTH_AUTHORITY_NAME'] + " (" + f.properties['HLTH_AUTHORITY_CODE'] + ")<br/>");
				},
		    error: function(xhr, err) {
		  	}
		  });	// end ajax call
		} // end lookupCHSAInfo function

		// Geocode Address autocomplete
		$('#geocodeField').autocomplete({
			minLength: 3,
			source: geocodeSuggest,
			select: function(evt, ui) {
				receiveGeocode(ui.item.data);
			}
		});

		var geocodeLayer = null;

		// Geocode Address event handler
		function geocode($field, callback, locationDescriptor = defaultLocationDescriptor) {
			var latLng = isCoords($field.val());
			if(latLng) {
				callback(latLng);
			} else {
				$.ajax({
					url: gcApi + "addresses.json",
					data: {
						matchPrecisionNot: matchPrecisionNot,
						locationDescriptor: locationDescriptor,
						echo: true,
						brief: true,
						addressString: $field.val()
					},
					success: callback,
					error: function(request) {
						alert(baseErrorMsg + "Error retrieving geocode results, please try your search again.");
						console.log(request);
					}
				});
			}
		}

		function isCoords(input) {
			var matches = input.match(/^\s*(-?\d{1,3}(?:\.\d*)?)\s*,\s*(-?[0-9]{1,3}(?:\.\d*)?)\s*$/);
			if(matches != null) {
				var lat = matches[1];
				var lng = matches[2];
				if(lat >= -90 && lat <= 90 && lng >= -180 && lng <= 180) {
					return new L.latLng(lat,lng);
				}
			}
			return false;
		}

		
		$('#geocodeField').keypress(function(e) {
 			if(e.which == 13) {
				if(geocodeLayer) {
					map.removeLayer(geocodeLayer);
					geocodeLayer = null;
				}
    		geocode($('#geocodeField'), receiveGeocode);
    		return false;
  		}
		});

		$('#geocodeBtn').on("click", function() {
			if(geocodeLayer) {
				map.removeLayer(geocodeLayer);
				geocodeLayer = null;
			}
			geocode($('#geocodeField'), receiveGeocode);
		});

		function setTextField(selector, value) {
			if(value !== undefined) {
				$(selector).val(value);
			}
		}

		function setCheckbox(selector, value) {
			if(value !== undefined) {
				if(value == "true" || value === true) {
					$(selector).prop('checked', true);
				} else {
					$(selector).prop('checked', false);
				}
			}
		}

		function setUrlParams(params) {
			var url = new URL(window.location.href);
			url.searchParams.set("routeParams", encodeURIComponent(JSON.stringify(params)));
			window.history.replaceState(null, document.title, url);
		}

		// Find Occupants By Name Autocomplete
		$('#nameField').autocomplete({
			minLength: 4,
			source: function(request, response) {
				$.ajax({
					url: gcApi + "occupants/addresses.json",
					data: {
						minScore: 50,
						maxResults: 5,
						echo: false,
						autoComplete: true,
						brief: true,
						addressString: request.term
					},
					success: function(data) {
						var list = [];
						if(data.features && data.features.length > 0) {
							list = data.features.map( function(item) {
								return {
									value: item.properties.fullAddress,
									data: item
								};
							});
						}
						response(list);
					},
					error: function() {
						response([]);
					}
				});
			},
			select: function(evt,ui) {
				if(namedOccLayer) {
					map.removeLayer(namedOccLayer);
				}
				namedOccLayer = L.geoJson(ui.item.data, {
					onEachFeature: function(feature, layer) {
						layer.bindPopup(makePopupText(feature.properties));
						layer.options['title'] = feature.properties.fullAddress;
					}
				}).addTo(map);
				centerMap(namedOccLayer.getBounds());
			}
		});

		var namedOccLayer;

		// Find Occupants By Name event handler
		function findOccs($field) {
			var addr = $field.val();
			$.ajax({
				url: gcApi + "occupants/addresses.json",
				data: {
					echo: 'false',
					brief: true,
					addressString: addr
				},
				success: function(data) {
					if(namedOccLayer) {
						map.removeLayer(namedOccLayer);
						namedOccLayer = null;
					}
					namedOccLayer = L.geoJson(data, {
						onEachFeature: function(feature, layer) {
							layer.bindPopup(makePopupText(feature.properties));
							layer.options['title'] = feature.properties.fullAddress;
						}
					}).addTo(map);
					centerMap(namedOccLayer.getBounds());
				},
				error: function(request) {
					alert(baseErrorMsg + "Error retrieving occupants by name, please try your search again.");
					console.log(request);
				}
			});
		};

		$('#nameField').keypress(function(e) {
			if(e.which == 13) {
				findOccs($('#nameField'));
				return false;
			}
		});

		$('#findOccsByName').on("click", function() {
			findOccs($('#nameField'));
		});

		$('#nameClear').on('click', function() {
			if(namedOccLayer) {
				map.removeLayer(namedOccLayer);
				namedOccLayer = null;
			}
		});

		function latLngToLayer(latLng, markerOptions) {
			var marker = L.marker(latLng, markerOptions);
			marker.bindPopup('<span class="popup-title">' + latLng.lat + "," + latLng.lng + '</span>');
			var layer = L.featureGroup([marker]);
			return layer;
		}

		function centerMap(bounds, center = true) {
			var options = {
				maxZoom: 16
			};
			if(center) {
				map.fitBounds(bounds.pad(0.25), options);
			} else if(!map.getBounds().contains(bounds.pad(0.25))) {
				// if the bounds aren't within the current map bounds
				// zoom out to include the bounds
				map.fitBounds(bounds.extend(map.getBounds()).pad(0.25), options);
			}
		}

		function date2String(date) {
			var tzOffset = -(date.getTimezoneOffset());
			return date.getFullYear() + '-'
				+ (date.getMonth()+1).toString().padStart(2, '0') + '-'
				+ (date.getDate()).toString().padStart(2, '0') + 'T'
				+ (date.getHours()).toString().padStart(2, '0') + ':'
				+ (date.getMinutes()).toString().padStart(2, '0') + ':'
				+ '00' + (tzOffset < 0 ? '-' : '+')
				+ Math.abs(tzOffset / 60).toString().padStart(2, '0') + ':'
				+ (tzOffset % 60).toString().padStart(2, '0');
		}

		if(queryParams.get('q')) {
			$('#geocodeField').val(queryParams.get('q'));
			geocode($('#geocodeField'), receiveGeocode);
		}

</script>
<template>
<div id="container">
	<!--<form> -->
	<div id="selectionContainer">
		<ul class="w3-ul w3-card-4">
			<li class="w3-bar">
				<!-- :disabled="disabled == 1" -->
				<input ref="search" v-model="search_input" type="text"/> <br/><br/>	<!-- search_input holds search query -->
				<span @click="filter('all')" class="button" v-bind:id="current['all']">All</span>
				<span @click="filter('Clear')" class="button" v-bind:id="current['Clear']">Clear weather</span>
			</li>
			<li class="w3-bar" v-bind:id="currentPath[path.id]" v-for="path in filteredList()" v-bind:key="path" @click="selected(path.id)">{{path.name}}</li>	<!-- get presented list from filteredList() method -->
			
		</ul>
	</div>
	<!--</form> -->
	<div id="mapContainer">
		<l-map id="map" ref="myMap"> </l-map>
	</div>


	<div class="container" id="resultsContainer">
        <form>

			<div class="row">
                <div class="col-50">
                    <label for="fname">Path Name</label>
                </div>
                <div class="col-50">
                    <input type="text" placeholder="Path name" readonly v-bind:value="path_name">
                </div>
            </div>

            <div class="row">
                <div class="col-50">
                    <label for="fname">Origin Stop name</label>
                </div>
                <div class="col-50">
                    <input type="text" placeholder="Origin stop name" readonly v-bind:value="origin_stop_name">
                </div>
            </div>

           <div class="row">
                <div class="col-50">
                    <label for="fname">Destination Stop name</label>
                </div>
                <div class="col-50">
                    <input type="text" placeholder="Destination stop name" readonly v-bind:value="destination_stop_name">
                </div>
            </div>

            <div class="row">
                <div class="col-50">
                    <label for="fname">Origin Coordinates</label>
                </div>
                <div class="col-25">
                    <input type="text" placeholder="Lat" readonly v-bind:value="origin_lat">
                </div>
                <div class="col-25">
                    <input type="text" placeholder="Lon" readonly v-bind:value="origin_lon">
                </div>
            </div>

            <div class="row">
                <div class="col-50">
                    <label for="fname">Destination Coordinates</label>
                </div>
                <div class="col-25">
                    <input type="text" placeholder="Lat" readonly v-bind:value="dest_lat">
                </div>
                <div class="col-25">
                    <input type="text" placeholder="Lon" readonly v-bind:value="dest_lon">
                </div>
            </div>

            <div class="row">
                <div class="col-50">
                    <label for="fname">Distance in meters</label>
                </div>
                <div class="col-50">
                    <input type="text" placeholder="Distance" readonly v-bind:value="distance">
                </div>
            </div>
            
            <div class="row">
                <div class="col-50">
                    <label for="fname">Next arrival at Origin</label>
                </div>
                <div class="col-50">
                    <input type="text" placeholder="Arrival" readonly v-bind:value="origin_next_arrival_at">
                </div>
            </div>

            

        </form>
        
    
        
    </div>


		
</div>

</template>

<script>
import regeneratorRuntime from "regenerator-runtime";
import L from 'leaflet';
import { LMap, LTileLayer, LMarker } from 'vue2-leaflet';

export default {
	components:{
		'l-map': LMap,
		'l-tile-layer': LTileLayer,
		'l-marker': LMarker
	},
	data () {
		return {
			
			// style
			current: [],
			currentPath: [],

			search_input: '',	// search bar
			
			// basic
			list:[],			// list of paths to show
			paths: [],			// holds list of paths (same with total but not indexed)
			clear_paths:[],		// holds list of clear paths
			mymap: {},
			weather_info:[],
			total:[],			// holds all paths, indexed by path id
			filtered: 'all',	// holds filtered state

			
			// search text
			disabled: 1,

			// path
			path:'',
			origin_stop_name:'',
			destination_stop_name:'',
			origin_lat:0,
			origin_lon:0,
			dest_lat:0,
			dest_lon:0,
			distance:0,
			origin_next_arrival_at:'',

			// map 
			src_marker:{},
			dst_marker:{},
			pathline:{},
			defaultIcon: L.icon({
                iconUrl: require('../assets/default.png'),
                iconSize: [58, 85],
                iconAnchor: [22, 94],
                popupAnchor: [-3, -76],
			}),
			
			// timeout
			handle:0,
			timeout:1000 * 60
		}
	},
	methods:{

		// SELECT PATH --> SET VIEW AND MAP
		// --------------------------------------------------------------------------------------------------------------------------------------------------
		selected(path_id){
			this.currentPath = [];
			this.currentPath[path_id] = 'selectedPath';
			// path information
			const path = this.total[path_id];
			this.path_name = path.name;
			this.origin_stop_name = path.origin_stop.name;
			this.destination_stop_name = path.destination_stop.name;
			this.origin_lat = path.origin_stop.lat;
			this.origin_lon = path.origin_stop.lon;
			this.dest_lat = path.destination_stop.lat;
			this.dest_lon = path.destination_stop.lon;
			this.distance = path.total_distance;
			this.origin_next_arrival_at = new Date(path.origin_stop.next_arrival);


			// markers
			if(this.src_marker !== {}) this.mymap.removeLayer(this.src_marker);
			if(this.dst_marker !== {}) this.mymap.removeLayer(this.dst_marker);

			this.src_marker = L.marker([this.origin_lat, this.origin_lon],{title:this.origin_stop_name})
				.setIcon(this.defaultIcon)
                .addTo(this.mymap);

			this.dst_marker = L.marker([this.dest_lat, this.dest_lon],{title:this.destination_stop_name})
				.setIcon(this.defaultIcon)
                .addTo(this.mymap);


			// polyline
			if(this.pathline !== {}) this.mymap.removeLayer(this.pathline);
			this.pathline = L.polyline(path.coordinates).addTo(this.mymap)
			this.mymap.fitBounds(this.pathline.getBounds());
			

		},
		// --------------------------------------------------------------------------------------------------------------------------------------------------

		
		// FILTERS
		// --------------------------------------------------------------------------------------------------------------------------------------------------
		
		// FILTER USING SEARCH QUERY
		filteredList(){
			return this.list.filter(path => {
        		return path.name.toLowerCase().includes(this.search_input.toLowerCase());
      		});
		},

		// FILTER USING WEATHER CONDITIONS
		filter(type){
			this.current = [];
			this.current[type] = 'selected';

			this.filtered=type;
			if(type === 'all') {
				this.list = this.paths;
			}
			else {
				this.list = this.clear_paths;
			}
		},
		// --------------------------------------------------------------------------------------------------------------------------------------------------

		// get all paths
		// --------------------------------------------------------------------------------------------------------------------------------------------------
		get_paths(weather){
			console.log(weather);
			// CORS
			let headers = {
				'Access-Control-Allow-Origin': 'http://localhost:8081/'
			};

			this.$http.get("http://localhost:8080/api/app2/getPaths/", {headers})
			.then((res) => {
				let i;
				this.paths = res.body;	// paths list
				this.clear_paths = [];	// clear weather paths
				for(i=0; i<this.paths.length; i++){
					this.total[this.paths[i].id] = this.paths[i];	// paths indexed list
					if(this.paths[i].origin_stop.weather === weather) this.clear_paths.push(this.paths[i]);  // keep clear weather origin stops here
				}
				
				if(this.filtered === 'all') this.list = this.paths;
				else this.list = this.clear_paths;
				console.log(this.clear_paths);
				console.log(this.list);
				
			

				console.log("CALLED");
			})
			
		}
		// --------------------------------------------------------------------------------------------------------------------------------------------------
	},
	mounted(){
		// create a map
		this.mymap = this.$refs.myMap.mapObject.setView([40.481426, 23.179408], 13);
		this.filtered = 'all';
		// tile for our map
		L.tileLayer('https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token=pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw', {
			maxZoom: 18,
			attribution: 'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, ' +
				'<a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
				'Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
			id: 'mapbox/streets-v11',
			tileSize: 512,
			zoomOffset: -1
		}).addTo(this.mymap);

		this.get_paths('Clear');
		if(this.handle === 0) this.handle = setInterval(this.get_paths, this.timeout, 'Clear');
	},
	destroyed(){
		while(this.handle > 0){
			clearTimeout(this.handle);
			this.handle--;
		}
	}
}
</script>

<style scoped>
	@import url('https://www.w3schools.com/w3css/4/w3.css');

	/* basic layout */
	ul{
		min-height: 100vh;	/* will enable overflow, will remain if nothing displayed */
	}
	/* UL and SELECTION CONTAINER will be 100% height and will float left */
	#selectionContainer{
		text-align: left;
		padding-left: 0px;	
		width: 30%;
		float: left;
		height: 100vh;
		overflow-y:scroll;
		min-height: 800px;
		
  	}

	/* mapContainer will float to the right */
	#mapContainer{
		width: 70%;		/* 70% to match selectionContainer */
		float: right;
	}

	/* map will get 50% of height */
	#map{
		height: 50vh;
		width: 100%;
	}
	   
	/* results will get the rest 50% */
	#resultsContainer{
		height: 50vh;
		border-radius: 5px;
		background-color: #f2f2f2;
		padding: 20px;
		float:right;
		width: 70%;	/* match selectionContainer */
	}

	li{
		margin-top:20px;
	}

	/* selected and selectedPath id's for selections highlight */
	#selected{
		background-color: rgb(59, 185, 175);
	}

	#selectedPath{
		border: 2px solid rgb(21, 226, 209);
	}


	input[type=text], select, textarea {
		width: 100%;
		padding: 10px;
		border: 1px solid #ccc;
		border-radius: 4px;
		resize: vertical;
	}
	
	.button {
		background-color: #4c74af; /* Green */
		border: none;
		color: white;
		padding: 15px 32px;
		text-align: center;
		text-decoration: none;
		display: inline-block;
		font-size: 16px;
		width: 200px;
		margin: 20px;
	}
	.button:hover{
		cursor: pointer;
	}

	li:hover{
		cursor: pointer;
	}



	/* cookie cutter */
	*{
		box-sizing: border-box;
	}


    label {
		padding: 10px 10px 10px 0;
		display: inline-block;
    }


    
    .col-25 {
		float: left;
		width: 25%;
		margin-top: 6px;
    }

    .col-50{
        float:left;
        width:50%;
        margin-top: 6px;
    }
    .col-75 {
		float: left;
		width: 75%;
		margin-top: 6px;
    }

    /* Clear floats after the columns */
    .row:after {
		content: "";
		display: table;
		clear: both;
    }


</style>



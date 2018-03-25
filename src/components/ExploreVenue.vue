<template>
  <div>
    <div class="user-input">
      Search Venue: <input type="text" v-model="query" class="query" placeholder="Enter venue you want to explore">
      Radius: <input type="text" v-model="radius" class="radius" placeholder="Enter radius to be covered">
      <button @click="exploreVenues">Explore</button>
    </div>
    <button id="photos-button" @click="showVenuePhotos">View Photos</button>
    <div class="explore-venues" v-if="dataLoadingCompleted">
      <div class="venue-on-list">
        <div class="venue-list-item" v-for="(item, key) in venuesList" :key="key">
          <div class="venue-details">
            <div class="sub-details">
              <span>{{item.venue_details.venue.name}}</span>
              <span v-show="item.venue_details.venue.rating">Ratings: {{item.venue_details.venue.rating}}</span>
            </div>  
            <span class="url"><a :href="item.venue_details.venue.url" target="_blank">{{item.venue_details.venue.url}}</a></span>
            <div class="venue-location">
              <span v-for="location in item.venue_details.venue.location.formattedAddress"
              :key="location">{{location}},</span>
            </div>
          </div>
          <div class="venue-photos">
            <span v-if="item.venue_photos.length === 0 && showPhotos">Sorry, no images available</span>
            <carousel-3d v-if="showPhotos && item.venue_photos.length" ref="mainCarousel" 
            :count="item.venue_photos.length" :controls-visible="true" :display=3 :height="120" :width="120">
              <slide v-for="(slide, i) in item.venue_photos" :index="i" :key=i>
                <figure>
                  <img class="photos" :src=slide>
                </figure>
              </slide>
            </carousel-3d>
          </div>
        </div>
      </div>
      <div class="venue-on-map" v-if="showMarkers">
        <gmap-map
          :center="center"
          map-type-id="terrain"
          :zoom= "12"
          class="map-size">
          <gmap-info-window :options="infoOptions" :position="infoWindowPosition"
          :opened="infoWindowOpen" @closeclick="infoWindowOpen=false">
            {{infoContent}}
          </gmap-info-window>
          <gmap-marker
            v-for="(m, index) in markers"
            :key="index"
            :position="m.position"
            :clickable="true"
            :draggable="true"
            @click="toggleInfoWindow(m,index)">
          </gmap-marker>
        </gmap-map>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import { Carousel3d, Slide } from 'vue-carousel-3d'
import * as VueGoogleMaps from 'vue2-google-maps'
import Vue from 'vue'

Vue.use(VueGoogleMaps, {
  load: {
    key: 'AIzaSyB5CE1jYbYfNyMukCPiqreymASfdRxAlUw'
  }
})

export default {
  name: 'ExploreVenue',
  props: {
    msg: String
  },
  components: {
    'carousel-3d': Carousel3d,
    'slide': Slide
  },
  data () {
    return {
      clientId: 'HOHQ2PJ1CQCYUHBYKP54R0UNEHHSHZVENJ0SPN5CEBBNKPVZ',
      clientSecret: 'NQ1WLVIZI0FLI5DXGCHM4WQKVDX5ID5O2Z5BOGHT5BPXFYPW',
      query: 'coffee',
      radius: 3000,
      position: null,
      latlong: null,
      venuesList: {},
      venuePhotos: {},
      dataLoadingCompleted: false,
      showPhotos: false,
      center: {lat: 10.0, lng: 10.0},
      infoContent: '',
      infoWindowPosition: {
        lat: 0,
        lng: 0
      },
      infoWindowOpen: false,
      currentIndex: null,
      // This is to assure that the infoWindow is visible on top of the marker
      infoOptions: {
        pixelOffset: {
          width: 0,
          height: -35
        }
      },
      markers: [{position: {lat: 0, lng: 0}, infoText: ''}],
      showMarkers: false
    }
  },
  beforeMount () {
    this.exploreVenues()
  },
  methods: {
    exploreVenues: function () {
      this.venuesList = {}
      this.markers = [{position: {lat: 0, lng: 0}, infoText: ''}]
      this.showMarkers = false
      this.dataLoadingCompleted = false
      this.showPhotos = false
      console.log('explore venues called')
      // TODO : when user denys persmission in the browser, what will happen then ?
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(position => {
          this.position = position.coords
          this.latlong = this.position.latitude + ',' + this.position.longitude
          this.center.lat = this.position.latitude
          this.center.lng = this.position.longitude
          this.getVenues()
        })
      }
    },

    getVenues: function () {
      axios.get('https://api.foursquare.com/v2/venues/explore', {
        params: {
          client_id: this.clientId,
          client_secret: this.clientSecret,
          ll: this.latlong,
          v: '20170801',
          query: this.query,
          limit: 10,
          intent: 'browse',
          radius: this.radius,
          group: 'venue'
        }
      }).then(response => {
        const items = response.data.response.groups[0].items
        let index = 0
        items.reduce((obj, item) => {
          this.venuesList[item.venue.id] = { 'venue_details': item }
          this.markers[index++] = {'position': {'lat': item.venue.location.lat, 'lng': item.venue.location.lng},
            'infoText': item.venue.name + ',' + item.venue.location.formattedAddress}
        })

        if (this.markers.length === Object.keys(this.venuesList).length) {
          this.showMarkers = true
        }

        for (let [ key, value ] of Object.entries(this.venuesList)) {
          value.venue_photos = this.getVenuePhotos(key)
        }
        this.dataLoadingCompleted = true
      })
    },

    getVenuePhotos: function (venueId) {
      let photos = []
      axios.get('https://api.foursquare.com/v2/venues/' + venueId + '/photos', {
        params: {
          client_id: this.clientId,
          client_secret: this.clientSecret,
          ll: this.latlong,
          v: '20170801',
          limit: 10,
          radius: this.radius
        }
      }).then(response => {
        for (let photo of response.data.response.photos.items) {
          photos.push(photo.prefix + '400x400' + photo.suffix)
        }
      })
      return photos
    },

    showVenuePhotos: function () {
      this.showPhotos = !this.showPhotos
      if (this.showPhotos) {
        document.getElementById('photos-button').innerHTML = "Hide Photos"
      }
      else {
        document.getElementById('photos-button').innerHTML = "View Photos"
      }
    },

    toggleInfoWindow: function (marker, index) {
      this.infoWindowPosition = marker.position
      this.infoContent = marker.infoText
      // If same marker is selected, then toggle
      if (this.currentIndex === index) {
        this.infoWindowOpen = !this.infoWindowOpen
      }
      // if different marker, then show the infoWindow
      else {
        this.infoWindowOpen = true
        this.currentIndex = index
      }
    }
  }
}
</script>

<style lang="scss">

.user-input {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
  margin-bottom: 50px;

  @media only screen and (max-width: 1024px) {
    justify-content: flex-start;
    align-items: flex-start;
  }

  @media only screen and (min-width: 360px) and (max-width: 500px) {
    flex-direction: column;
    justify-content: flex-start;
    align-items: flex-start;
  }

  >input {
    height: 40px;
    margin-right: 20px;
    padding-left: 10px;
    font-size: 14px;
    color: gray;
    margin-left: 10px;
    border: 1px solid gray;

    &.query {
      width: 350px;

      @media only screen and (min-width: 768px) and (max-width: 1024px) {
        width: 250px;
      }

      @media only screen and (min-width: 360px) and (max-width: 500px) {
        width: 320px;
        margin-bottom: 20px;
      }
    }

    &.radius {
      width: 200px;

      @media only screen and (min-width: 360px) and (max-width: 500px) {
        width: 320px;
        margin-bottom: 20px;
      }
    }
  }
}

.explore-venues {
  display: flex;
  flex-direction: row;
  padding: 10px;
  margin-top: 30px;

  @media only screen and (max-width: 1024px) {
    flex-direction: column;
  }

  .venue-on-list {
    flex: 0 1 50%;
    @media only screen and (max-width: 1024px) {
      flex: 0 0 100%;
    }
  }

  .venue-on-map {
    flex: 0 1 50%;
    box-shadow: 0 3px 1px -2px rgba(0,0,0,.2), 0 2px 2px 0 rgba(0,0,0,.14), 0 1px 5px 0 rgba(0,0,0,.12);
    border-radius: 2px;
    transition: .3s cubic-bezier(.4,0,.2,1);
    padding: 15px 10px;
    max-height: fit-content;

    @media only screen and (max-width: 992px) {
      flex: 0 1 100%;
    }

    @media only screen and (min-width: 993px) and (max-width: 1200px) {
        flex: 0 1 40%;
        margin-left: 30px;
    }

    .map-size {
      width: 750px; 
      height: 500px;
      @media only screen and (min-width: 360px) and (max-width: 500px) {
        width: 320px; 
        height: 450px;
      }
      @media only screen and (min-width: 501px) and(max-width: 768px) {
        width: 705px; 
        height: 450px;
      }
      @media only screen and (min-width: 769px) and (max-width: 992px) {
        width: 350px; 
        height: 800px;
      }
      @media only screen and (min-width: 993px) and (max-width: 1024px) {
        width: 930px; 
        height: 800px;
      }
      @media only screen and (min-width: 1025px) and (max-width: 1200px) {
        width: 600px; 
        height: 800px;
      }
    }
  }
}

.venue-list-item {
  display: flex;
  flex-direction: row;
  margin-bottom: 40px;
  flex-wrap: wrap;

  @media only screen and (min-width: 360px) and (max-width: 1200px) {
    flex-direction: column;
  }

  .venue-details {
    flex: 0 0 40%;
    padding: 10px 25px;

    @media only screen and (min-width: 768px) {
      min-width: 450px;
    }
  }

  .venue-photos {
    flex: 0 0 40%;
    display: flex;
    flex-direction: row;
    @media only screen and (max-width: 768px) {
      margin-left: 50px;
    }
    @media only screen and (min-width: 992px) {
      margin-left: 70px;
    }
  }
}


.venue-details {
  display: flex;
  flex-wrap: wrap;
  flex-direction: column;
  box-shadow: 0 3px 1px -2px rgba(0,0,0,.2), 0 2px 2px 0 rgba(0,0,0,.14), 0 1px 5px 0 rgba(0,0,0,.12);
  border-radius: 2px;
  transition: .3s cubic-bezier(.4,0,.2,1);
  background-color: #e9e9e9;
  color: black;

  .sub-details {
    flex: 0 0 auto;
    display: flex;
    flex-direction: row;
    justify-content: space-between;

    >span {
      font-weight: bold;
      &:first-child {
        color: black;
      }
      &:last-child {
        color: red;
      }
    }
  }

  .url {
    text-align: left;
    &:hover {
      cursor: pointer;
      text-decoration: underline;
      color: blue;
    }
  }
}

.venue-location {
    flex: 0 1 auto;
    display: flex;
    flex-direction: row;
    

    @media only screen and (max-width: 768px) {
      flex-direction: column;
      align-items: flex-start;
      justify-content: flex-start;
    }

    @media only screen and (min-width: 360px) and (max-width: 500px) {
      flex-direction: row;
      flex-wrap: wrap;
    }

    >span {
      flex: 0 0 auto;
      margin-right: 5px;
    }
  }

.venue-photos {
  .photos {
    width: 150px;
    height: 180px;
    padding-left: -21px;
    margin-left: -44px;
    margin-top: -30px;
  }

  .more-photos {
    display: none;
  }

  .single-photo {
    &:hover {
      .more-photos {
        display: block;
      }
    }
  }
}

.carousel-3d-container {
  margin-top: 30px !important;
  margin-bottom: 0px !important;
}

button {
    color: white;
    background: gray;
    padding: 13px 0;
    width: 250px;
    border: 2px solid transparent;
    border-radius: 4px;
    font-weight: 700;
    transition: all 0.3s ease-in-out;
    cursor: pointer;
    font-size: .875rem;
    text-transform: uppercase;
    display: block;

    &#photos-button {
      margin-left: 5px;
    }

    &:hover {
      color: gray;
      background: white;
      border-color: gray;
    }

    @media only screen and (min-width: 360px) and (max-width: 500px) {
      display: block;
      margin: 0 auto;
    }
  } 
</style>

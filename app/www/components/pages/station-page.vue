<template>
    <div>
        <toolbar></toolbar>
        <div class="content-within-page">
            <md-card v-if="station.ge_id">
                <vueper-slides v-if="loadedPhotos" autoplay :infinite="false" class="station-slides">
                    <vueper-slide v-for="(photo, index) in station.photos" :key="index">
                        <div slot="slideContent">
                            <img class="station-photo" :style="{backgroundImage: convertPhoto(photos[photo.id])}">
                        </div>
                    </vueper-slide>
                </vueper-slides>
                <md-card-media-cover md-solid>
                    <md-card-area>
                        <md-card-header>
                            <span class="md-title">{{ station.name }}</span>
                            <span class="md-subhead">{{ station.address.street }} <br> {{ station.address.postcode }}
                                {{ station.address.city }} <br> {{ station.address.country }}</span>
                            <div class="station-network-operator">
                                {{ station.network }} ({{ station.operator }})
                            </div>
                            <div class="station-status">
                                <img v-if="station.verified && !station.fault_report" src="icons/verified.svg">
                                <img v-if="!station.verified && !station.fault_report" src="icons/unverified.svg">
                                <b>
                                    <span v-if="station.verified && !station.fault_report">{{ translated.VERIFIED }}</span>
                                    <span v-if="!station.verified && !station.fault_report">{{ translated.UNVERIFIED }}</span>
                                </b>
                                <img v-if="station.fault_report" src="icons/error.svg">
                                <b>
                                    <span v-if="station.fault_report">{{ translated.MALFUNCTION }}</span>
                                </b>
                            </div>
                        </md-card-header>
                    </md-card-area>
                </md-card-media-cover>
                <md-card-content>
                    <h3 class="md-subheading">{{ translated.PLUGS }}</h3>
                    <ul>
                        <li v-for="(chargepoint, index) in station.chargepoints" :key="index">
                            {{ chargepoint.count }}x {{ chargepoint.type }} [{{ chargepoint.power }}kW]
                        </li>
                    </ul>
                    <div class="md-subhead">
                        <div v-if="station.cost.freecharging" class="status-overview-icon">
                            <img src="icons/money_off.svg">
                            <b><span>{{ translated.FREE_CHARGING }}</span></b>
                        </div>
                        <div v-if="station.cost.freeparking" class="status-overview-icon">
                            <img src="icons/parking.svg">
                            <b><span>{{ translated.FREE_PARKING }}</span></b>
                        </div>
                        <div v-if="station.barrierfree" class="status-overview-icon">
                            <img src="icons/accessibility.svg">
                            <b><span>{{ translated.BARRIERFREE }}</span></b>
                        </div>
                        <div v-if="station.openinghours['24/7']" class="status-overview-icon">
                            <img src="icons/access_time.svg">
                            <b><span>24/7</span></b>
                        </div>
                    </div>
                    <div style="clear: both"></div>
                    <b>{{ translated.GENERAL }}:</b><br>{{ station.general_information || '-' }}<br>
                    <b>{{ translated.PASTIME }}:</b><br>{{ station.ladeweile || '-' }}<br>
                    <b>{{ translated.LOCATION_DESCRIPTION }}:</b><br>{{ station.location_description || '-' }}<br>
                </md-card-content>
                <md-card-content>
                    <h3 class="md-subheading">{{ translated.COSTS }}</h3>
                    <b>{{ translated.SHORT_DESCRIPTION }}:</b><br>{{ station.cost.description_short || '-'}}<br>
                    <b>{{ translated.DESCRIPTION }}:</b><br>{{ station.cost.description_long || '-' }}<br>
                </md-card-content>
                <md-card-content>
                    <h3 class="md-subheading">{{ translated.CHARGE_CARDS }}</h3>
                    <span v-if="!station.chargecards.length">-</span>
                    <ul>
                        <li v-for="(card, index) in station.chargecards" :key="index">
                            {{ getCard(card.id) }}
                        </li>
                    </ul>
                </md-card-content>
                <md-card-content>
                    <h3 class="md-subheading">{{ translated.OPENING_HOURS }}</h3>
                    <b>{{ translated.DESCRIPTION }}:</b><br>{{ station.openinghours.description || '-' }}<br>
                    <div v-if="station.openinghours['24/7']">
                        <img src="icons/access_time.svg">
                        <b><span>24/7</span></b>
                    </div>
                    <div v-if="!station.openinghours['24/7']">
                        <div v-for="(day, index) in Object.keys(station.openinghours.days)" :key="index">
                            <span><b>{{ translated[day.toUpperCase()] }}:</b> {{
                                formatOpeningTime(station.openinghours.days[day]) }}</span>
                        </div>
                    </div>
                </md-card-content>
                <md-card-content>
                    <div id="station-map" ref="map"></div>
                </md-card-content>
                <md-card-content>
                    {{ translated.SOURCE }}: <a href="https://goingelectric.de">GoingElectric.de</a>
                </md-card-content>
            </md-card>
        </div>
        <bottom-bar></bottom-bar>
    </div>
</template>

<script>
    import http from './../modules/http.vue';
    import eventBus from './../modules/event.vue';
    import translation from './../modules/translation.vue';
    import toolbar from './../container/toolbar.vue';
    import bottomBar from './../container/bottom-bar.vue';

    export default {
        data() {
            return {
                station: {},
                translated: {},
                photos: {},
                loadedPhotos: false
            };
        },
        watch: {
            'station.photos': 'getPhotos'
        },
        methods: {
            formatOpeningTime(time) {
                if (time === 'closed') return this.translated.CLOSED;
                return time.replace('from', '').replace('till', '').replace('  ', '-');
            },
            getStation() {
                var self = this;

                // get station details
                http.sendRequest('get', 'station', {
                    id: self.$route.query.id
                }, true, (err, station) => {
                    if (!err && station) {
                        self.station = station;
                        setTimeout(() => window.scrollTo(0, 0), 200);
                        // wait for cordova device to be ready - apply listener, if not ready yet
                        eventBus.$off('deviceReady');
                        if (self.$root.deviceReady) self.buildMap();
                        else {
                            eventBus.$on('deviceReady', function () {
                                self.buildMap();
                            });
                        }
                    } else {
                        // TODO
                    }
                });
            },
            getCard(id) {
                try {
                    return this.$root.stationcards.filter(card => card.card_id === id)[0].name;
                } catch (e) {
                    return '?';
                }
            },
            getPhotos() {
                var self = this,
                    cnt = 0;

                self.station.photos.forEach(photo => {
                    self.$http.get(RESTURL + 'stationphoto', {
                        params: {
                            id: photo.id
                        },
                        responseType: 'arraybuffer'
                    }).then(response => {
                        self.photos[photo.id] = response.body;
                        if (++cnt === self.station.photos.length) self.loadedPhotos = true;
                    }).catch(); // TODO
                });
            },
            convertPhoto(photo) {
                var binary = '',
                    bytes = new Uint8Array(photo);

                // convert the binary
                for (var i = 0; i < bytes.byteLength; i++) binary += String.fromCharCode(bytes[i]);
                return 'url("data:image/png;base64,' + btoa(binary) + '")';
            },
            buildMap() {
                var self = this;

                if (self.station.coordinates != null && self.station.coordinates.lat && self.station.coordinates.lng) {
                    if (!window.plugin || !plugin.google || !plugin.google.maps) return;
                    Vue.nextTick(() => {
                        var map = plugin.google.maps.Map.getMap(document.getElementById('station-map')),
                            maxPower = Math.max.apply(Math, (self.station.chargepoints.map(chargepoint => chargepoint.power))),
                            icon = ((maxPower >= 50) ? 'fast' : ((maxPower >= 11)? 'normal' : 'slow'));

                        map.moveCamera({
                            target: {
                                lat: self.station.coordinates.lat,
                                lng: self.station.coordinates.lng
                            },
                            zoom: 17
                        });
                        // build the marker for the station
                        map.addMarker({
                            position: {
                                lat: self.station.coordinates.lat,
                                lng: self.station.coordinates.lng
                            },
                            icon: 'icons/ev_station_' + icon + '.svg', // determine speed to set color
                            title: self.station.name,
                            snippet: self.station.address.street + '<br>' + self.station.address.postcode +
                                ' ' +
                                self.station.address.city + '<br>' + self.station.address.country,
                            animation: plugin.google.maps.Animation.BOUNCE
                        });
                    });
                }
            }
        },
        components: {
            toolbar,
            bottomBar
        },
        mounted() {
            var self = this;

            self.translated = translation.translatePage();
            if (self.$route.query.id) {
                eventBus.$off('station_openInNew');
                eventBus.$on('station_openInNew', () => {
                    if (self.station.url) window.open('https:' + self.station.url, '_blank');
                });
                eventBus.$off('station_navigate');
                eventBus.$on('station_navigate', () => {
                    if (self.station.coordinates && window.cordova && window.launchnavigator) {
                        launchnavigator.navigate([self.station.coordinates.lat, self.station.coordinates.lng]);
                    }
                });
                eventBus.$off('stationcardsCached');
                // retrieve station after cards has been cached
                if (self.$root.stationcards.length) self.getStation();
                else eventBus.$on('stationcardsCached', () => self.getStation());
            }
        }
    }
</script>

<style scoped>
    .vueperslides {
        background-color: #4b4b4b;
    }

    .md-card-media-cover {
        margin-top: 140px;
    }

    .station-network-operator {
        position: absolute;
        left: 16px;
        bottom: 8px;
        font-style: italic;
        font-size: 11px;
    }

    .station-status {
        position: absolute;
        right: 16px;
        bottom: 16px;
    }

    .status-overview-icon {
        float: left;
    }

    .station-photo {
        min-height: 140px;
        background-repeat: no-repeat;
        background-size: contain;
        width: 100vw;
        background-position: center center;
    }
</style>

<style>
    .vueperslides__arrows .vueperslides__arrow {
        fill: #448aff;
    }

    #station-map {
        width: 100%;
        height: 500px;
    }
</style>
<template>
    <section class="ymap-container">
        <div :id="ymapId" :style="{ width: '100%', height: '100%' }"></div>
        <div id="baloon-helper">
            <slot name="baloon"></slot>
        </div>
        <slot></slot>
    </section>
</template>

<script>
    export default {
        data() {
            return {
                ymapId: 'yandexMap' + Math.round(Math.random()*100000)
            }
        },
        props: {
            coords: {
                type: Array,
                validator(val) {
                    return !val.filter(item => isNaN(item)).length
                },
                required: true
            },
            zoom: {
                validator(val) {
                    return !isNaN(val)
                },
                default: 18
            }
        },
        computed: {
            coordinates() {
                return this.coords.map(item => +item)
            }
        },
        beforeCreate() {
            if (this.$ymapEventBus.scriptIsNotAttached) {
                const yandexMapScript = document.createElement('SCRIPT');
                yandexMapScript.setAttribute('src', 'https://api-maps.yandex.ru/2.0/?load=package.full&lang=ru-RU');
                yandexMapScript.setAttribute('async', '');
                yandexMapScript.setAttribute('defer', '');
                document.body.appendChild(yandexMapScript);
                this.$ymapEventBus.scriptIsNotAttached = false;
                yandexMapScript.onload = () => {
                    this.$ymapEventBus.ymapReady = true;
                    this.$ymapEventBus.$emit('scriptIsLoaded');
                }
            } else {
                return false;
            }
        },
        created() {
	        window.addEventListener('DOMContentLoaded', () => {
                let myMap;

                if (this.$ymapEventBus.ymapReady) {
                    ymaps.ready(init.bind(this));
                } else {
                    this.$ymapEventBus.$on('scriptIsLoaded', () => {
                        ymaps.ready(init.bind(this));
                    })
                }

                function init() {
                    var addMarkers = function() {
                        if (!self.$slots || !self.$slots.default) {
                            return;
                        }
                        if (self.$slots.default) {
                            const myMarkers = self.$slots.default.map(marker => {
                                const props = marker.componentOptions && marker.componentOptions.propsData;
                                if (!props) return;
                                return {
                                    markerType: props.markerType,
                                    coords: setCoordsToNumeric(props.coords),
                                    hintContent: props.hintContent,
                                    icon: props.icon,
                                    balloon: props.balloon,
                                    markerStroke: props.markerStroke,
                                    markerFill: props.markerFill,
                                    circleRadius: +props.circleRadius
                                }
                            }).filter(marker => marker && marker.markerType);

                            for (let i = 0; i < myMarkers.length; i++) {
                                const markerType = setFirstLetterToUppercase(myMarkers[i].markerType);
                                const properties = {
                                    hintContent: myMarkers[i].hintContent,
                                    balloonContentHeader: myMarkers[i].balloon && myMarkers[i].balloon.header,
                                    balloonContentBody: myMarkers[i].balloon && myMarkers[i].balloon.body,
                                    balloonContentFooter: myMarkers[i].balloon && myMarkers[i].balloon.footer,
                                    iconContent: myMarkers[i].icon && myMarkers[i].icon.content,
                                };
                                const options = {
                                    preset: myMarkers[i].icon && `islands#${getIconPreset(myMarkers[i])}Icon`,
                                    strokeColor: myMarkers[i].markerStroke && myMarkers[i].markerStroke.color || "0066ffff",
                                    strokeOpacity: myMarkers[i].markerStroke && myMarkers[i].markerStroke.opacity || 1,
                                    strokeStyle: myMarkers[i].markerStroke && myMarkers[i].markerStroke.style,
                                    strokeWidth: myMarkers[i].markerStroke && myMarkers[i].markerStroke.width || 1,
                                    fill: myMarkers[i].markerFill && myMarkers[i].markerFill.enabled || true,
                                    fillColor: myMarkers[i].markerFill && myMarkers[i].markerFill.color || "0066ff99",
                                    fillOpacity: myMarkers[i].markerFill && myMarkers[i].markerFill.opacity || 1
                                };
                                if (markerType === 'Circle') {
                                    myMarkers[i].coords = [myMarkers[i].coords, myMarkers[i].circleRadius];
                                }
                                const marker = new ymaps[markerType](myMarkers[i].coords, properties, options);
                                myMap.geoObjects.add(marker);
                            }
                        }
                    }

                    var addBaloon = function() {
                        if (!self.$slots || !self.$slots.baloon) {
                            return;
                        }

                        var baloon = self.$slots.baloon.map(slot => {
                            var MyBalloonContentLayoutClass = ymaps.templateLayoutFactory.createClass(
                                '<div id="baloon" class="baloon">'
                                + document.getElementById("baloon-helper").innerHTML
                                + '</div>'
                            );
                            ymaps.layout.storage.add('my#baloon', MyBalloonContentLayoutClass);

                            var myPlacemark = new ymaps.Placemark(
                                slot.data.attrs.coords,
                                {
                                    iconImageOffset: [0, 0]
                                },
                                {
                                    balloonContentLayout: MyBalloonContentLayoutClass,
                                }
                            );
                            myPlacemark.options.set({
                                balloonShadow: false
                            });

                            var myCollection = new ymaps.GeoObjectCollection();
                            myCollection
                                .add(myPlacemark)

                            myCollection.options.set({
                                balloonContentBodyLayout: 'my#baloon',
                                balloonMaxWidth: 700
                            });

                            myMap.controls.add('zoomControl');

                            myMap.geoObjects.add(myCollection);
                            myPlacemark.balloon.open();
                        });
                    }

                    var self = this;

                    myMap = new ymaps.Map(self.ymapId, {
                        center: self.coordinates,
                        zoom: +self.zoom
                    });

                    addMarkers();
                    addBaloon();

                }

                function getIconPreset(marker) {
                    let firstPart = marker.icon.color || 'blue',
                        secondPart;
                    if (marker.icon.glyph) {
                        secondPart = setFirstLetterToUppercase(marker.icon.glyph);
                    } else if (marker.icon.content) {
                        secondPart = 'Stretchy'
                    } else {
                        secondPart = ''
                    }
                    return firstPart + secondPart
                }

                function setFirstLetterToUppercase(string) {
                    return string.charAt(0).toUpperCase() + string.slice(1);
                }

                function setCoordsToNumeric(arr) {
                    return arr.map(item => {
                        return Array.isArray(item) ? setCoordsToNumeric(item) : +item;
                    })
                }
            })
        }
    }
</script>

<style scoped>
#baloon-helper {
    display: none;
}
</style>

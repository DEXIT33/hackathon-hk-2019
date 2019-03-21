<template>
<div class="container-main" v-bar="{
  preventParentScroll: true
}" ref="scrollbar">
  <div @scroll="handleScroll" ref="contentContainer">
    <v-container class="container-header">
      <h1 class="display-3 font-weight-black ">Všechny akce</h1>
      <p class="title font-weight-regular">Řazeno od dnešního data</p>
    </v-container>
    <v-container class="container-content" ref="content">
      <masonry
        :cols="cols"
        gutter="30"
        >
        <event-card
          v-for="event in events"
          v-if="event"
          :key="event.id"
          :eventId="event.id"
          :text="event.anotation"
          :title="event.name"
          :date="formatDate(event.date_time_start_first.toDate())"
          :place="places.has(String(event.place)) ? places.get(String(event.place)).name : ''"
          :img="(event.photos && event.photos.length > 0) ? 'http://www.hkregion.cz/galerie/obrazky/imager.php?img=' + event.photos[0] + '&x=500' : undefined"></event-card>
      </masonry>
    </v-container>
  </div>
</div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import EventCard from '../components/EventCard.vue';
import Firebase from 'firebase';

@Component({
  components: {
    EventCard,
  },
})
export default class Home extends Vue {

  private columns = 4;
  private events: any[] = [];
  private places = new Map<string, any>();

  private loading = false;
  private lastLimit = 0;
  private step = 20;
  private lastDate: Date = new Date();

  private created() {
    window.addEventListener('resize', this.handleResize);

    this.lastLimit += this.step;
    Firebase.firestore()
      .collection('events')
      .orderBy('date_time_start_first')
      .startAt(Firebase.firestore.Timestamp.fromDate(new Date()))
      .limit(this.lastLimit)
      .onSnapshot((snapshot) => {
        this.loadPlaces(snapshot).then(() => {
          for (let i = 0; i < this.step; i++) {
            this.$set(this.events, i, snapshot.docs[i].data());
            this.events[i].id = snapshot.docs[i].id;
            this.lastDate = snapshot.docs[i].data().date_time_start_first;
          }
        });
      });
  }

  private loadNext() {
    this.loading = true;

    this.lastLimit += this.step;
    Firebase.firestore()
      .collection('events')
      .orderBy('date_time_start_first')
      .startAt(this.lastDate)
      .limit(this.lastLimit)
      .onSnapshot((snapshot) => {
        this.loadPlaces(snapshot).then(() => {
          for (let i = this.lastLimit - this.step; i < this.step; i++) {
            this.$set(this.events, i, snapshot.docs[i].data());
            this.events[i].id = snapshot.docs[i].id;
            this.lastDate = snapshot.docs[i].data().index;
          }
          this.loading = false;
        });
      });
  }

  private mounted() {
    this.handleResize();
  }

  private formatDate(date: Date) {
    // tslint:disable-next-line:max-line-length
    return date.getDate() + '.' + date.getMonth() + '.' + date.getFullYear() + ' ' + date.getHours() + ':' + (date.getMinutes() === 0 ? '00' : date.getMinutes());
  }

  private handleResize() {
    if ((this.$refs.content as Element).clientWidth < 960) {
      this.columns = 1;
    } else if ((this.$refs.content as Element).clientWidth < 1280) {
      this.columns = 2;
    } else if ((this.$refs.content as Element).clientWidth < 1690) {
      this.columns = 3;
    } else {
      this.columns = 4;
    }
  }

  private handleScroll() {
    this.handleResize();

    // @ts-ignore
    if ((this.$refs.contentContainer as Element).clientHeight - 100 < this.$vuebar.getState(this.$refs.scrollbar).barTop + this.$vuebar.getState(this.$refs.scrollbar).barHeight) {
      if (!this.loading) {
        this.loadNext();
      }
    }
  }

  private async loadPlaces(snapshot: Firebase.firestore.QuerySnapshot) {
    const promises = [];

    for (const doc of snapshot.docs) {
      if (!this.places.has(String(doc.data().place))) {
        promises.push(Firebase.firestore().collection('places').doc(String(doc.data().place)).get());
      }
    }

    const results = await Promise.all(promises);
    results.forEach((place) => {
      if (!this.places.has(String(place.id))) {
        this.places.set(String(place.id), place.data());
      }
    });
  }

  private destroyed() {
    window.removeEventListener('resize', this.handleResize);
  }

  get cols() {
    return this.columns;
  }
}
</script>

<style lang="scss">
.container-main {
  height: calc(100vh - 64px) !important;
  height: calc(calc(var(--vh, 1vh) * 100) - 64px);

  @media screen and (max-width: 959px) {
    height: calc(100vh - 56px) !important;
    height: calc(calc(var(--vh, 1vh) * 100) - 56px);
  }

  width: 100%;
  max-width: initial;
}

.container-header {
  padding-top: 50px;
  padding-left: 70px;
  padding-right: 70px;
  max-width: initial;

  @media screen and (max-width: 959px) {
    padding-left: 25px;
    padding-right: 25px;
  }
}

.container-content {
  padding-left: 70px;
  padding-right: 70px;
  max-width: initial;

  @media screen and (max-width: 959px) {
    padding-left: 25px;
    padding-right: 25px;
  }
}

.display-3 {
  font-family: "LatoLatinWebHeavy" !important;
  font-weight: bold !important;
}
</style>
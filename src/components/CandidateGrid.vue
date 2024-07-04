<template>
  <v-container>
    <v-app-bar app>
      <v-btn :icon="!soundEnabled ? 'mdi-volume-off' : 'mdi-volume-high'"  @click="toggleSound"></v-btn>
      <v-app-bar-title>Election Information <span class="ml-4 text-subtitle-2 text-disabled">Data from Democracy Club</span></v-app-bar-title>

        <v-spacer></v-spacer>
      <v-app-bar-items>

        <v-btn color="secondary" :prepend-icon="bigBeastsFilter ? 'mdi-shark-fin' : 'mdi-shark-fin-outline'" @click="toggleBigBeastsFilter" :variant="bigBeastsFilter ? 'flat' : 'outlined'">Big Beasts Only</v-btn>

        <v-spacer></v-spacer>
      </v-app-bar-items>
      <v-btn>{{ unelectedCount }} out of {{ totalCount }} candidates are detonated</v-btn>

    </v-app-bar>
    <audio ref="explosionSound" src="/exploding-tories/explosion.mp3" preload="auto"></audio>
    <v-virtual-scroll :items="chunkedCandidates" :item-height="320" height="800">
      <template v-slot:default="{ item: candidatesChunk }">
        <v-row>
          <v-col v-for="candidate in candidatesChunk" :key="candidate.person_id" cols="12" md="4">
            <v-card
              :class="{ 'exploding': candidate.isExploding, 'unelected': candidate.isUnelected, 'elected': candidate.isElected }"
              @animationend="handleAnimationEnd(candidate)"
            >
              <v-card-title>{{ candidate.person_name }}</v-card-title>
              <v-card-subtitle>{{ candidate.ballot_paper_id }}</v-card-subtitle>
              <v-card-text>
                <div>Election ID: {{ candidate.election_id }}</div>
                <div>Elected: {{ candidate.elected }}</div>
                <div v-if="candidate.image">
                  <v-img
                    :src="candidate.image"
                    height="200px"
                    contain
                    alt="Candidate image"
                    class="grey lighten-2 opacity-80"
                  ></v-img>
                </div>
                <div v-else>
                  <v-avatar size="200" class="grey lighten-2">
                    <v-icon size="100">mdi-account</v-icon>
                  </v-avatar>
                </div>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </template>
    </v-virtual-scroll>
  </v-container>
</template>

<script>
import axios from 'axios';

export default {
  name: 'CandidateGrid',
  data() {
    return {
      candidates: [],
      oldCandidatesState: {},
      soundEnabled: false,
      bigBeastsFilter: false, // State of the big beasts filter toggle
      bigBeasts: new Set(['308', '2595', '451', '2346', '4176', '6138', '5777', '197', '1692', '769', '5464', '3155', '188', '1612', '5371', '4223', '3938', '1481', '6499', '3267', '5031' ]) // List of big beasts candidate IDs
    };
  },
  computed: {
    unelectedCount() {
      return this.filteredCandidates.filter(candidate => candidate.elected === 'f').length;
    },
    totalCount() {
      return this.filteredCandidates.length;
    },
    filteredCandidates() {
      if (this.bigBeastsFilter) {
        return this.candidates.filter(candidate => this.bigBeasts.has(candidate.person_id));
      }
      return this.candidates;
    },
    chunkedCandidates() {
      const chunkSize = 3;
      const chunks = [];
      for (let i = 0; i < this.filteredCandidates.length; i += chunkSize) {
        chunks.push(this.filteredCandidates.slice(i, i + chunkSize));
      }
      return chunks;
    }
  },
  mounted() {
    this.pollCandidates();
    setInterval(this.pollCandidates, 10000);  // Poll every 10 seconds
  },
  methods: {
    toggleBigBeastsFilter() {
      this.bigBeastsFilter = !this.bigBeastsFilter;
    },
    pollCandidates() {
      axios.get('https://api.pawsey.press/election-data')
        .then(response => {
          this.updateCandidates(response.data);
        })
        .catch(error => console.error('Error fetching candidates:', error));
    },
    updateCandidates(newCandidates) {
      newCandidates.forEach(candidate => {
        const oldCandidate = this.oldCandidatesState[candidate.person_id] || {};
        const wasUnelected = oldCandidate.elected === 'f';

        // Update current state based on new data
        candidate.isUnelected = candidate.elected === 'f';
        candidate.isElected = candidate.elected === 't';

        // Check if there was a change to 'unelected' and it has not exploded yet
        if (!wasUnelected && candidate.isUnelected && !oldCandidate.hasExploded) {
          this.triggerExplosion(candidate);
        }
      });

      // Update the oldCandidatesState with the latest data to prevent re-triggering
      this.oldCandidatesState = newCandidates.reduce((acc, curr) => {
        acc[curr.person_id] = { ...curr, hasExploded: curr.hasExploded };
        return acc;
      }, {});
      this.candidates = newCandidates;
    },
    triggerExplosion(candidate) {
      if (!candidate.hasExploded) {
        console.log("Exploding for:", candidate.person_name);
        candidate.isExploding = true;
        candidate.hasExploded = true; // Mark as exploded to prevent future triggers
        if (this.soundEnabled && this.filteredCandidates.some(c => c.person_id === candidate.person_id)) {
          this.playExplosionSound();
        }
      }
    },
    toggleSound() {
      this.soundEnabled = !this.soundEnabled;
      console.log("Sound toggled:", this.soundEnabled);
    },
    enableSound() {
      this.soundEnabled = true;
      console.log("Sound enabled:", this.soundEnabled);
    },
    playExplosionSound() {
      const explosionSound = this.$refs.explosionSound;
      explosionSound.play().catch(error => {
        console.error('Error playing sound:', error);
      });
    },
    handleAnimationEnd(candidate) {
      if (candidate.isExploding) {
        candidate.isExploding = false;
      }
      this.$forceUpdate();
    }
  }
};
</script>


<style scoped>
@keyframes explode {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.5);
    opacity: 0.5;
  }
  100% {
    transform: scale(1) rotate(-10deg);
    opacity: 0;
  }
}

.exploding {
  animation: explode 1s forwards; /* Ensures the animation completes to the final state */
}

.unelected {
  transform: rotate(-10deg) scale(0.9); /* Ensures the card stays wonky after the animation */
  opacity: 0.333;
}
</style>

<template>
    <div>
      <h1>Counter-Strike Scoreboard</h1>
  
      <!-- File Input -->
      <div>
        <input type="file" @change="handleFileUpLoad">
      </div>
  
      <!-- Loading Indicator -->
      <div v-if="loading" class="text-center">Parsing file...</div>
  
      <!-- Scoreboard -->
      <div v-else-if="players.length" class="container">
        <table class="table">
          <thead>
            <tr>
              <th>Player</th>
              <th>Kills</th>
              <th>Deaths</th>
              <th>K/D ratio</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="player in players.filter(p => p.team === 'CT')">
              <td>{{ player.name }}</td>
              <td>{{ player.kills }}</td>
              <td>{{ player.deaths }}</td>
              <td>{{ player.kills / player.deaths }}</td>
            </tr>
          </tbody>
        </table>
        <table class="table">
          <thead>
            <tr>
              <th>Player</th>
              <th>Kills</th>
              <th>Deaths</th>
              <th>K/D ratio</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="player in players.filter(p => p.team === 'TERRORIST')">
              <td>{{ player.name }}</td>
              <td>{{ player.kills }}</td>
              <td>{{ player.deaths }}</td>
              <td>{{ player.kills / player.deaths }}</td>
            </tr>
          </tbody>
        </table>
      </div>
      
  
      <!-- No Data Message -->
      <div v-else>
        <p class="text-center text-gray-500">No data available. Please upload a valid log file.</p>
      </div>
    </div>

    <div>
      <h2>Stats per round</h2>
      <select id="roundSelector" v-model.number="selectedRoundNumber">
        <option value="" disabled>Select a key</option>
        <option v-for="key in keys" :key="key" :value="key">
          {{ key }}
        </option>
      </select>
      
      <div v-if="selectedRound !== null" class="container">
        <table class="table">
          <thead>
            <tr>
              <th>Player</th>
              <th>Kills</th>
              <th>Deaths</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="player in selectedRound.filter(p => p.team === 'CT')">
              <td>{{ player.name }}</td>
              <td>{{ player.kills }}</td>
              <td>{{ player.deaths }}</td>
            </tr>
          </tbody>
        </table>
        <table class="table">
          <thead>
            <tr>
              <th>Player</th>
              <th>Kills</th>
              <th>Deaths</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="player in selectedRound.filter(p => p.team === 'TERRORIST')">
              <td>{{ player.name }}</td>
              <td>{{ player.kills }}</td>
              <td>{{ player.deaths }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </template>
  
  <script lang="ts">
  import { defineComponent } from 'vue';
  
  interface PlayerStats {
    name: string;
    team: string;
    kills: number;
    deaths: number;
  }
  
  export default defineComponent({
    name: 'Scoreboard',
    data() {
      return {
        players: [] as PlayerStats[],
        loading: false,
        rounds: new Map<number, Map<string, PlayerStats>>,
        selectedRoundNumber: null as number | null
      };
    },

    computed: {
      selectedRound(): PlayerStats[] | null {
        if (this.selectedRoundNumber !== null && this.rounds.has(this.selectedRoundNumber)) {
          return Array.from(this.rounds.get(this.selectedRoundNumber)!.values());
        }
        return null
      },

      keys(): number[] {
        return Array.from(this.rounds.keys());
      }
    },

    methods: {
        async handleFileUpLoad(event: Event) {
            const file = (event.target as HTMLInputElement).files?.[0];
            if (!file) return;

            this.loading = true;
            this.players = []; // Reset previous data

            // Read file content
            const reader = new FileReader();
            reader.onload = (e) => {
              const content = e.target?.result as string;
              if (content) {
                this.players = this.parseLogFile(content); // Parse log content
              }
              this.loading = false;
            };
            reader.readAsText(file);
        },
  
        // Parse Log File Content
        parseLogFile(content: string): PlayerStats[] {
          const logs = content.split('\n');
          var stats = new Map<string, PlayerStats>;

          var roundStats = new Map<number, Map<string, PlayerStats>>;
          var currentRound = 0;
    
          logs.forEach((line) => {
              if (line.includes('World triggered "Match_Start"')) {
                stats.forEach(player => {
                  player.deaths = 0;
                  player.kills = 0;
                });

                roundStats = new Map<number, Map<string, PlayerStats>>;
                currentRound = 0;
              }

              if (line.includes('World triggered "Round_Start"')) {
                currentRound++;
                const roundPlayerStats = new Map<string, PlayerStats>;
                stats.forEach(player => {
                  roundPlayerStats.set(player.name, { name: player.name, team: player.team, kills: 0, deaths: 0 })
                });
                roundStats.set(currentRound, roundPlayerStats);
              }

              const teamSwitchMatch = line.match(/"([^<]+)<\d+><STEAM_[^>]+>" switched from team <([^>]+)> to <([^>]+)>/);
              if(teamSwitchMatch) {
                const [, playerName, , team] = teamSwitchMatch;
                if (team != 'Unassigned' && team != 'Spectator') {
                  if(!stats.has(playerName)) {
                    stats.set(playerName, { name: playerName, team: team, kills: 0, deaths: 0 });
                  }
                  if(!roundStats.get(currentRound)!.has(playerName)) {
                    roundStats.get(currentRound)!.set(playerName, { name: playerName, team: team, kills: 0, deaths: 0 });
                  }
                }
              }
              
              // Count kills and deaths
              if (line.includes('killed')) {
                const killMatch = line.match(/"([^<]+)<\d+><STEAM_[^>]+><([^>]+)>".*"([^<]+)<\d+><STEAM_[^>]+><([^>]+)>"/);
                if (killMatch) {
                  const [,killerName, , victimName, ] = killMatch;
                  stats.get(killerName)!.kills++;
                  stats.get(victimName)!.deaths++;

                  roundStats.get(currentRound)!.get(killerName)!.kills++;
                  roundStats.get(currentRound)!.get(victimName)!.deaths++;
                }
              }
          });

          this.rounds = new Map(roundStats);
          return Array.from(stats.values()).sort((p1, p2) => p2.kills - p1.kills);
        },
    },
  });
  </script>
  <style>
    body {
      font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    }

    h1, h2 {
      font-style: italic;
      text-transform: uppercase;
    }

    .container {
      display: flex;
      gap: 16px;
    }

    .table {
      width: 50%;
    }

    table {
      margin: 20px auto;
    }

    th,
    td {
      font-size: 19px;
      border: 2px solid rgb(42 24 37);
      padding: 20px;
    }

    th {
      font-weight: bold;
    }

    thead {
      background-color: rgb(42 24 37);
    }
  </style>
  
<template>
  <div>
    <div class="notification">
      <button @click="callAlert"><img src="../assets/3602156.png" alt="Alert" /></button>
      <p><small>Press for notification demo </small></p>
    </div>
    <div class="hello">
      <h1>Search Vaccination Slots</h1>
      <audio id="chatAudio">
        <source src="https://media.geeksforgeeks.org/wp-content/uploads/20190531135120/beep.mp3" 
        type="audio/mpeg">
      </audio>
      <div class="wrapper">
        <div class="flex-row">
          <div class="flex-col-6">
            <a
              :class="{'button-active': showPostalForm}"
              class="button-style"
              @click="setFormType(true)"
            >Search By PIN</a>
          </div>
          <div class="flex-col-6">
            <a
              :class="{'button-active': !showPostalForm}"
              class="button-style"
              @click="setFormType(false)"
            >Search By District</a>
          </div>
        </div>
        <form @submit="submitForm" v-on:submit.prevent>
          <div v-if="showPostalForm">
            <input
              type="text"
              class="flex-col-6"
              placeholder="Enter your Pin"
              v-model="pinCode"
            >

            <span class="error" v-show="errors.length && pincodeError">{{ pincodeError.pincode }}</span>
          </div>
          <div v-else>
            <div class="flex-row">
              <div class="flex-col-6">
                <select
                  v-model="state"
                  @input="selectedState($event.target.value)"
                  @blur="selectedState($event.target.value)"
                >
                  <option
                    v-for="(state, index) in stateList"
                    :key="index"
                    :value="state.state_id"
                  >
                    {{ state.state_name }}
                  </option>
                </select>

                <span class="error" v-show="errors.length && stateError">{{ stateError.state }}</span>
              </div>
              <div class="flex-col-6">
                <select
                  v-model="district"
                >
                  <option
                    v-for="(state, index) in districtList"
                    :key="index"
                    :value="state.district_id"
                  >
                    {{ state.district_name }}
                  </option>
                </select>
                <span class="error" v-show="errors.length && districtError">{{ districtError.district }}</span>
              </div>
            </div>
          </div>
          <div class="flex-row">
            <input type="submit" class="search-btn" value="Submit">
          </div>
        </form>
      
        <div v-if="centersFound.length">
          <hr>
          <div class="flex-row">
            <table>
              <thead>
                <tr>
                  <th>Center Name</th>
                  <th>Fee Type</th>
                  <th>Count</th>
                  <th>Date</th>
                  <th>Slots Time</th>
                </tr>
              </thead>
              <tr
                v-for="(row, index) in centersFound"
                :key="index"
              >
                <td> {{ row.name }}</td>
                <td> {{ row.fee_type }}</td>
                <td width="10%">
                  <div v-for="(slot, index) in row.sessions" :key="index">
                    {{ slot.available_capacity }}
                  </div>
                </td>
                <td width="10%">
                  <div v-for="(slot, index) in row.sessions" :key="index">
                    {{ slot.date }}
                  </div>

                </td>
                <td>
                  <div v-for="(slot, index) in row.sessions" :key="index">
                    <span  v-for="(time, index) in slot.slots" :key="index">
                      {{ time }}, 
                    </span>
                  </div>

                </td>
              </tr>
            </table>
          </div>
        </div>
        <div v-if="noSlotsFound">
          <hr>
          <h3>Oops No Center found! We would keep searching in background in every 15 minutes, You would hear a beep once the slot is available.</h3>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import moment from 'moment';

export default {
  name: 'Cowin',

  data() {
    return {
      centersFound: [],
      errors: [],
      district: '',
      districtDefault: [{
        district_id: '',
        district_name: 'Select District',
      }],
      districtError: '',
      districtResp: [],
      pinCode: '',
      pincodeError: '',
      showPostalForm: true,
      noSlotsFound: false,
      state: '',
      stateError: '',
      statesResp: [],
      stateDefault: [{
        state_id: '',
        state_name: 'Select State',
      }],
    };
  },

  computed: {
    districtList() {
      return [...this.districtDefault, ...this.districtResp];
    },
    stateList() {
      return [...this.stateDefault, ...this.statesResp];
    },
  },

  watch: {
    errors(val) {
      if (val.length) {
        if (this.showPostalForm) {
          this.pincodeError = val.find(err => err.pincode);
        }
        else {
          this.districtError = val.find(err => err.district) || '';
          this.stateError = val.find(err => err.state) || '';
        }
      }
    },
  },

  methods: {
    callAlert() {
      let audio = document.getElementById('chatAudio');
      
      audio.loop = true;
      audio.play();

      setTimeout(() => {
        audio.loop = false;
      }, 3 * 1000); // Repeat for 3 sec
    },
    centerFilter(val) {
      let searchInterval;

      this.centersFound = [];
      
      val.forEach(center => {
        center.sessions.forEach(session => {
          if (session.available_capacity && session.min_age_limit === 18 && session.slots.length) {
            clearInterval(searchInterval);
            this.centersFound.push(center);

            this.callAlert();
          }

          else {
            this.noSlotsFound = true;
            searchInterval = setInterval(() => {
              this.showPostalForm ? this.onPincodeEnter(this.pinCode) : this.selectedDistrict(this.district);
            }, 15 * 60 * 1000); // Interval of 15 mins
          }
        });
      });

      this.centersFound = [...new Set(this.centersFound)];
    },

    onPincodeEnter(val) {
      const currentDate  = moment(new Date()).format('DD-MM-YYYY');

      axios
        .get(`https://cdn-api.co-vin.in/api/v2/appointment/sessions/public/calendarByPin?pincode=${val}&date=${currentDate}`)
        .then((response) => {
          this.centerFilter(response.data.centers);
        });
    },
    
    setFormType(data) {
      this.showPostalForm = data;

      if (!data) {
        axios
          .get('https://cdn-api.co-vin.in/api/v2/admin/location/states')
          .then((response) => {
            this.statesResp = response.data.states;
          });
      }
    },

    selectedState(val) {
      if (val.length) {
        axios
          .get(`https://cdn-api.co-vin.in/api/v2/admin/location/districts/${val}`)
          .then((response) => {
            this.districtResp = response.data.districts;
          });
      }
    },

    selectedDistrict(val) {
      if (val.length) {
        const currentDate  = moment(new Date()).format('DD-MM-YYYY');
        
        axios
          .get(`https://cdn-api.co-vin.in/api/v2/appointment/sessions/public/calendarByDistrict?district_id=${val}&date=${currentDate}`)
          .then((response) => {
            this.centerFilter(response.data.centers);
          });
      }
    },

    submitForm(e) {
      this.noSlotsFound = false;
      this.errors = [];
      
      if (this.pinCode) {
        const regex = /^[1-9][0-9]{5}$/;

        if (regex.test(this.pinCode)) {
          this.onPincodeEnter(this.pinCode);
        }

        else {
          this.errors.push({pincode: 'Invalid Pincode.'});
        }

        return true;
      }

      if (this.district && this.state) {
        this.selectedDistrict(this.district);

        return true;
      }


      if (this.showPostalForm) {
        if (!this.pinCode) {
          this.errors.push({pincode: 'Pincode is required.'});
        }
      }
      else {
        if (!this.state) {
          this.errors.push({state: 'State is required.'});
        }
        
        if (!this.district) {
          this.errors.push({district: 'District is required.'});
        }
      }
      
      e.preventDefault();
    }
  },
}
</script>
<style scoped>
.hello {
  display: block;
  flex: 1;
  overflow: auto;
  max-width: 80%;
  margin: 0 auto;
  height: 100%;
}
.wrapper {
  display: block;
  margin: 0 auto;
  width: 100%;
  float: none;
}
.flex-row {
  display: flex;
  flex-direction: row;
  padding: 10px;
  margin: 10px 0;
  align-items: center;
  align-content: center;
}
.flex-col-12 {
  flex: 0 0 90%;
  max-width: 90%;
}
.flex-col-6 {
  flex: 0 0 50%;
  max-width: 50%;
}
.flex-col-4 {
  flex: 0 0 40%;
  max-width: 0%;
}
h1 {
  margin: 40px 0;
}
a.button-style {
  background-color:#9CC3D5FF;
  padding: 10px 15px;
  color: #0063B2FF;
  font-weight: 500;
  margin: 0 5px;
}
a.button-active {
  background-color:#0063B2FF;
  color: #FFF;
  box-shadow: 4px 0 10px 0 rgba(0, 0, 0, 0.5);
}

.search-btn {
  font-weight: 500;
  background-color:#EFEFEF;
  padding: 10px 15px;
  color: #002060;
  border: 2px solid #002060;
  font-weight: 500;
  margin: 0 auto;
  float: none;
  width: 100px;
}

.search-btn:focus,
.search-btn:hover {
  box-shadow: 4px 0 10px 0 rgba(0, 0, 0, 0.5);
}

.error {
  font-size: 12px;
  font-weight: 900;
  color: red;
  min-height: 16px;
}
label {
  text-align: left;
  font-weight: 700;
}
input, select {
  border: 1px solid #9CC3D5FF;
  padding: 10px 15px;
  margin: 5px;
  width: 95%;
}
.notification {
  text-align: right;
}
.notification button {
  width: 40px;
  height: 35px;
}

.notification button img {
  max-width: 25px;
  height: auto;
}

.notification p {
  color: #e74c3c;
  font-weight: 900;
  margin: 5px 0;
}

table {
  width: 100%;
  border-collapse: collapse;
}
th, td {
  text-align: left;
  border-bottom: 1px solid #d2d2d7;
  padding: 6px;
}

th:last-of-type, td:last-of-type {
  text-align: right;
}

tr:last-of-type td{
  border: 0;
}
</style>

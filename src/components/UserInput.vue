<template>
  <div class="title-container">
    <h1 class="title">{{ msg }}</h1>
  </div>
  <div class="container">
    <div class="box">
        <div class="input-box">
            <div class="user-input">
              <label for="symbolInput" class="input-label">Symbol</label>
              <input class="symbol-input" v-model="symbol" type="text" placeholder="Enter symbol name..." name="symbolInput" />
            </div>
            <p class="error-msg" v-if="errorMessage">{{ errorMessage }}</p>
        </div>
        <div class="dropdown-wrapper">
            <label for="symbolInput" class="input-label">Period</label>
            <select v-model="selectedPeriod">
                <option value="daily">Daily</option>
                <option value="hourly">Hourly</option>
            </select>         
        </div>
    </div>
    
    <button 
      @click="fetchData" 
      class="fetch-btn"  
      :style="{ backgroundColor: isDisabled ? '#4CAF50' : '#006400',  opacity: isDisabled ? 0.6 : 1, cursor: isDisabled ?  'not-allowed' : 'pointer' }" 
      :disabled="isDisabled">
      Fetch Data
    </button>
    
    <!-- Loader -->
    <div v-if="loading" class="loader"></div>
    <h3 class="error" v-if="catchError">{{ catchError }}</h3>
    <!-- Display fetched data here -->
    <div class="chart" id="container" ref="chartContainer" v-if="showChart" style="width: 100% ; height: 400px;"></div>
  </div>

</template>

<script>
import { createChart } from 'lightweight-charts';

export default {
    name: 'UserInput',
    props: {
        msg: String
    },
    data() {
        return {
          // All req. data properties
          // Possible error handling messages
          errorMessages: {
            emptySymbol: 'Symbol cannot be empty',
            notValid: 'Invalid symbol name',
            networkError: 'Network not available. Please check your connection.',
            apiError: 'Failed to fetch data. Please try again later.'
          },
          // instances of the properties in data object
          symbol: '',
          selectedPeriod: 'daily',
          errorMessage: '',
          flagValue: false,
          isDisabled: false,
          isZoomed: false,
          showChart: true,
          catchError: false,
          loading: false
        };
    },
    watch: {
    symbol: {
      handler() {
        this.catchError = false;
        this.checkEmpty();
      },
    },
    selectedPeriod: {
      handler() {
        this.catchError = false;
        this.checkEmpty();
      },
    }
    },
    methods: {
        checkEmpty() {
          // Condition to check if the input field is empty
          // Disbale the button of fetching data from the server
            if (this.symbol) {
                this.isDisabled = false;
                this.errorMessage = '';
            } else {
                this.isDisabled = true;
                this.errorMessage = this.errorMessages.emptySymbol;
            }

            // Clear the chart container before rendering the new chart
            this.clearChartContainer();
        },
        clearChartContainer() {
            
            const chartContainer = document.getElementById('container');

            while (chartContainer.firstChild) {
                chartContainer.removeChild(chartContainer.lastChild);
            }
        },
        fetchData() {

          // Check network availability
          if (!window.navigator.onLine) {
            this.catchError = this.errorMessages.networkError;
            return;
          }

          // check if any input field is empty
            this.checkEmpty();

            // Run contition only if fetch button is enabled
            if (!this.isDisabled){

                // Clear the chart container before rendering the new chart
                this.clearChartContainer(); 

                // Enabling the loader for the waiting response from api
                this.loading = true;

                const url = `http://localhost:8081/api/search?symbol=${this.symbol}&period=${this.selectedPeriod}`;

                // fetching the url to get api data
                fetch(url)
                    .then(response => {
                    if (!response.ok) {
                      this.catchError = this.errorMessages.networkError;
                        throw new Error('Network response was not ok');
                    }
                    return response.json();
                    })
                    .then(data => {

                      // If the response was successful and the data was returned from the server
                      // then disable loading feature.
                        this.loading = false;

                        // Handle the received data
                        let timeSeriesData;

                        // Defining the data according to their time period
                        if(this.selectedPeriod === 'daily'){
                            timeSeriesData = data['Time Series (Daily)'];
                        }else if(this.selectedPeriod === 'hourly'){
                            timeSeriesData = data['Time Series (60min)'];
                        }
                    
                        // Once the data is fetched, set showChart to true to display the chart
                        this.showChart = true;

                        // Call the mountChart method with the formatted chart data
                        this.mountChart(timeSeriesData); 
                    
                })
                .catch(error => {
                    // handle error and return error message
                     console.log();(error.message);
                    this.catchError =this.errorMessages.apiError;
                    this.errorMessage = this.errorMessages.notValid;

                    // Clear the chart container before rendering the new chart
                    this.clearChartContainer();
                });
            }
        },
        mountChart(timeSeriesData) {

          // Take ref of div container where chart is getting displayed
            const chartDiv = document.getElementById('container');
            // Adding some styling to the chart
            const chartOptions = { layout: { textColor: 'black', background: { type: 'solid', color: 'white' } } };
                
            // Create the chart instance
            const chart = createChart(chartDiv, chartOptions);

            // Make sure the chart is responsive with resizing
            window.onresize = function() {
                chart.applyOptions({ 
                    width: chartDiv.offsetWidth, 
                    height: chartDiv.offsetHeight ,
                    handleScroll: true,
                    handleScale: true
                });
            }

            // Drawing candleStick chart pattern
            // Passing different colors to candle sticks
            const candlestickSeries = chart.addCandlestickSeries({
                upColor: '#26a69a', downColor: '#ef5350', borderVisible: false,
                wickUpColor: '#26a69a', wickDownColor: '#ef5350',
            });

            // Getting data in required format to create the chart
            const data = Object.keys(timeSeriesData).map(date => {
                const item = timeSeriesData[date];
                // Convert date to UNIX timestamp
                const formattedDate = new Date(date).getTime()/1000; 

                return {
                time: formattedDate,
                open: parseFloat(item['1. open']),
                high: parseFloat(item['2. high']),
                low: parseFloat(item['3. low']),
                close: parseFloat(item['4. close']),
                volume: parseFloat(item['5. volume'])
                };
        
            });

            // sorting the api resonose data in chronological order as lightweight-chart librabry's requirement
            const sortedData = data.sort((a, b) => a.time - b.time);

            // Setting the sorted data to create a new chart
            candlestickSeries.setData(sortedData);

            // Setting the time scale
            const timeScale = chart.timeScale();

            // Setting the time scale options gracefully for better user experience
            timeScale.applyOptions({
                borderColor: "#71649C",
                barSpacing: 15,
                minBarSpacing: 5,
                fixLeftEdge: true,
                fixRightEdge: true
            })

            // setting the visible state of the chart.
            timeScale.setVisibleRange({ from: sortedData[0].time, to: sortedData[99].time });

        }
    }
}
</script>


<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
/* css for styling */
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding-top: 25px;
}

.title-container {
  background-color: green;
}
.title {
  text-align: left;
  font-size: calc(1.2rem + 0.5vw);
  margin: 0;
  padding: 15px 30px;
}
.box {
  display: flex;
  gap: 30px;
  margin: 20px auto 10px auto;
  justify-content: center;
}

.user-input {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: flex-start;
}

label {
  font-size: 14px;
  padding-bottom: 5px;
}
.error-msg {
    text-align: left;
    color: red;
    font-size: 12px;
    margin-top: 5px;
}

.symbol-input {
  outline: none !important;
  box-shadow: none !important;
  height: 30px;
  border: 1px solid lightgreen;
  font-size: 16px;
  padding-left: 8px;
  border-radius: 5px;
}

.dropdown-wrapper{
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

select{
    -webkit-appearance: none;
    -moz-appearance: none;
    appearance: none;
    height: 33px !important;
    width: 150px;
    cursor: pointer;
    background-color: #fff;
    color: grey;
    background-image: url("arrow-down.png");
    background-repeat: no-repeat;
    background-size: 30px;
    background-position: right center;
    outline: none !important;
    border: 1px solid lightgreen;
    font-size: 16px;
    padding-left: 8px;
    border-radius: 5px;
    box-shadow: 0 0 20px rgba(20,20,30,0.25);
}
select::-ms-expand{
    display: none;
}

select option:hover {
    background-color: transparent; /* Set the background color to transparent */
}
select option{
    border: 1px solid lightgray;
    letter-spacing: 1.2px;
    font-weight: 400;
    font-size: 16px;
    padding: 10px 20px;
}
/* For Firefox */
select option:checked {
    background-color: initial; /* Set the background color to initial */
}
.selected{
    display: none;
}

.fetch-btn {
  outline: none !important;
  box-shadow: none !important;
  border: none;
  padding: 10px 15px;
  color: #fff;
  border-radius: 5px;
  margin: 20px 10px;
  font-size: 16px;
  letter-spacing: 0.5px;
  font-weight: 600;
}

.chart {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 20px;
}

.loader {
    border: 4px solid #f3f3f3;
    border-top: 4px solid #3498db;
    border-radius: 50%;
    width: 30px;
    height: 30px;
    animation: spin 2s linear infinite;
    margin: 20px auto;
  }

  .error {
    display: flex;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

/* Mobile Styles */
@media (max-width: 450px) {
  .container {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: flex-start;
    margin: 10px 10px 0 10px;
    padding: 10px 0;
  }

  .title {
    padding: 15px;
  }
  .box {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    width: 100%;
    gap: 30px;
  }

  .input-box {
    width: 90%;
  }
  .symbol-input{
    width: 100%;
  }

  .dropdown-wrapper {
    width: 70%;
  }

  select {
    width: 100%;
  }
  .fetch-btn {
    margin: 0px;
    margin-top: 20px;
  }
  .chart {
    margin-top: 20px;
  }

}

</style>

<script>
    import { onMount } from 'svelte';
    import Chart from 'chart.js';

    onMount(() => {

        //fetch the data
        Promise.all([
            fetch('https://shakepay.github.io/programming-exercise/web/rates_CAD_BTC.json').then((res) => res.json()),
            fetch('https://shakepay.github.io/programming-exercise/web/rates_CAD_ETH.json').then((res) => res.json()),
            fetch("https://shakepay.github.io/programming-exercise/web/transaction_history.json").then((res) => res.json()),
        ]).then(([btcRates, ethRates, transactionHistory]) => {

            //normalize currency rates by day
            let btcRatesDate = {};
            btcRates.forEach((rate) => {
                btcRatesDate[rate.createdAt.substring(0, 10)] = rate.midMarketRate;
            })

            let ethRatesDate = {};
            ethRates.forEach((rate) => {
                ethRatesDate[rate.createdAt.substring(0, 10)] = rate.midMarketRate;
            })

            //networth of each currency at a specific date
            let nw = {
                BTC: {},
                CAD: {},
                ETH: {},
                total: {},
            }
            let prevBal = {
                BTC: 0,
                CAD: 0,
                ETH: 0
            }
            transactionHistory.reverse().forEach((transaction) => {
                //add/subtract amount from previous balance
                if(transaction.type != 'conversion'){
                    if(transaction.direction == 'credit')
                        nw[transaction.currency][transaction.createdAt.substring(0, 10)] = prevBal[transaction.currency] + transaction.amount;
                    else{
                        nw[transaction.currency][transaction.createdAt.substring(0, 10)] = prevBal[transaction.currency] - transaction.amount;
                    }
                    prevBal[transaction.currency] = nw[transaction.currency][transaction.createdAt.substring(0, 10)] 
                }
                //subtract from one currency and add to the other
                else if (transaction.type == 'conversion'){
                    nw[transaction.from.currency][transaction.createdAt.substring(0, 10)] = prevBal[transaction.from.currency] - transaction.from.amount;
                    nw[transaction.to.currency][transaction.createdAt.substring(0, 10)] = prevBal[transaction.to.currency] + transaction.to.amount;
                    prevBal[transaction.from.currency] = nw[transaction.from.currency][transaction.createdAt.substring(0, 10)] 
                    prevBal[transaction.to.currency] = nw[transaction.to.currency][transaction.createdAt.substring(0, 10)] 
                }
            })

            let days = {};
            let prevDay = {
                BTC: 0,
                ETH: 0,
                CAD: 0,
            }

            //calculate the total networth on each date
            Object.keys(btcRatesDate).forEach((index) => {
                if(!days[index]){
                    days[index] = 0;
                }
                if(nw.CAD[index]){
                    days[index] += nw.CAD[index];
                    prevDay.CAD = nw.CAD[index];
                }
                else{
                    days[index] += prevDay.CAD
                }
                if(nw.BTC[index]){
                    days[index] += nw.BTC[index] * btcRatesDate[index];
                    prevDay.BTC = nw.BTC[index] * btcRatesDate[index];
                }
                else{
                    days[index] += prevDay.BTC;
                }
                if(nw.ETH[index]){
                    days[index] += nw.ETH[index] * ethRatesDate[index];
                    prevDay.ETH = nw.ETH[index] * ethRatesDate[index]
                }
                else{
                    days[index] += prevDay.ETH;
                }
                days[index] = days[index].toFixed(2);
            }); 
            initChart(days);
        }).catch((error) => {
        });
    })

    //chart.js chart
    const initChart = (data) => {
    const ctx = document.getElementById('networth-chart').getContext('2d');
    const chart = new Chart(ctx, {
        type: 'line',
        data: {
            labels: Object.keys(data),
            datasets: [{
                label: 'Networth Over Time',
                lineTension: 0,
                fill: false,
                pointRadius: 0,
                data: Object.values(data),
            }]
        },
        options: {
            scales: {
                xAxes: [{
                    type: 'time',
                    time: {
                        unit: 'month'
                    }
                }]
            },
            spanGaps: true
        }
    });
}
</script>

<div>
    <canvas id="networth-chart"></canvas>
</div>

<style>
    div{
        width: 75%;
        height: 75%;
    }
</style>
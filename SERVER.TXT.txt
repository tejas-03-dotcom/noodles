const express = require('express');
const axios = require('axios');
const cors = require('cors');

const app = express();
app.use(cors());

function selectOptionType(strikePrice, marketPrice) {
    return strikePrice < marketPrice ? "Call" : "Put";
}

app.get('/api/option-chain', async (req, res) => {
    const { symbol, expiry } = req.query;

    if (!symbol || !expiry) {
        return res.status(400).json({ error: 'Symbol and expiry date are required.' });
    }

    try {
        const response = await axios.get(`https://www.nseindia.com/api/option-chain-indices?symbol=${symbol}`, {
            headers: {
                'User-Agent': 'Mozilla/5.0',
                'Accept-Language': 'en-US,en;q=0.9',
                'Referer': 'https://www.nseindia.com/'
            }
        });

        const records = response.data.records;
        if (!records || !records.data) {
            return res.status(500).json({ error: 'Invalid API response.' });
        }

        const expiryDate = new Date(expiry).toLocaleDateString('en-GB', {
            day: '2-digit',
            month: 'short',
            year: 'numeric'
        }).replace(/ /g, '-');

        const filteredData = records.data.filter(item => item.expiryDate === expiryDate);
        const marketPrice = records.underlyingValue;

        const optionData = filteredData.map(item => ({
            strikePrice: item.strikePrice,
            marketPrice: marketPrice,
            optionType: selectOptionType(item.strikePrice, marketPrice),
            call: {
                LTP: item.CE ? item.CE.lastPrice : 0,
                OI: item.CE ? item.CE.openInterest : 0
            },
            put: {
                LTP: item.PE ? item.PE.lastPrice : 0,
                OI: item.PE ? item.PE.openInterest : 0
            }
        }));

        res.json(optionData);
    } catch (error) {
        console.error('Error fetching data:', error);
        res.status(500).json({ error: 'Failed to fetch data from NSE API.' });
    }
});

const PORT = 3000;
app.listen(PORT, () => console.log(`Server running at http://localhost:${PORT}`));

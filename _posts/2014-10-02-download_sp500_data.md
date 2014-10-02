---
layout: post
title: Downloading S&P 500 tickers and data using Python
---

Finding and dowloading a list of current S&P 500 companies and their respective price data can be tedious at best. Luckily for you, today I'm going to share with you a Python script that I use to construct a database of daily bar data for the current S&P 500 companies.

The script works as follows:

- scrape S&P 500 ticker list and industry sectors from [list of S&P 500 companies](https://en.wikipedia.org/wiki/List_of_S%26P_500_companies) on Wikipedia.
- download daily close data for each industry sector from [Yahoo finance](http://finance.yahoo.com) using pandas [DataReader](http://pandas.pydata.org/pandas-docs/stable/remote_data.html).
- adjust the open, high and low data using the ratio of the adjusted close to close.
- store the complete dataset in a local HDF5 file indexed by industry sector.

You can change the start and end dates using the `START` and `END` variables at the top of the scipt.

Recently, I've been investigating methods for constructing an S&P 500 dataset that is free of [survivorship bias](https://en.wikipedia.org/wiki/Survivorship_bias) but haven't succeeded. If you have any ideas or suggestions please leave a comment below or [contact me](/contact).

{% highlight python %}
#!/usr/bin/env python

import urllib2
import pytz
import pandas as pd

from bs4 import BeautifulSoup
from datetime import datetime
from pandas.io.data import DataReader


SITE = "http://en.wikipedia.org/wiki/List_of_S%26P_500_companies"
START = datetime(1900, 1, 1, 0, 0, 0, 0, pytz.utc)
END = datetime.today().utcnow()


def scrape_list(site):
    hdr = {'User-Agent': 'Mozilla/5.0'}
    req = urllib2.Request(site, headers=hdr)
    page = urllib2.urlopen(req)
    soup = BeautifulSoup(page)

    table = soup.find('table', {'class': 'wikitable sortable'})
    sector_tickers = dict()
    for row in table.findAll('tr'):
        col = row.findAll('td')
        if len(col) > 0:
            sector = str(col[3].string.strip()).lower().replace(' ', '_')
            ticker = str(col[0].string.strip())
            if sector not in sector_tickers:
                sector_tickers[sector] = list()
            sector_tickers[sector].append(ticker)
    return sector_tickers


def download_ohlc(sector_tickers, start, end):
    sector_ohlc = {}
    for sector, tickers in sector_tickers.iteritems():
        print 'Downloading data from Yahoo for %s sector' % sector
        data = DataReader(tickers, 'yahoo', start, end)
        for item in ['Open', 'High', 'Low']:
            data[item] = data[item] * data['Adj Close'] / data['Close']
        data.rename(items={'Open': 'open', 'High': 'high', 'Low': 'low',
                           'Adj Close': 'close', 'Volume': 'volume'},
                    inplace=True)
        data.drop(['Close'], inplace=True)
        sector_ohlc[sector] = data
    print 'Finished downloading data'
    return sector_ohlc


def store_HDF5(sector_ohlc, path):
    with pd.get_store(path) as store:
        for sector, ohlc in sector_ohlc.iteritems():
            store[sector] = ohlc


def get_snp500():
    sector_tickers = scrape_list(SITE)
    sector_ohlc = download_ohlc(sector_tickers, START, END)
    store_HDF5(sector_ohlc, 'snp500.h5')


if __name__ == '__main__':
    get_snp500()
{% endhighlight %}


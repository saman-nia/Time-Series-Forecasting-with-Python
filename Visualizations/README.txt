About the data
--------------

The two files contain electricity market history over several days.

order-books-20190521-20190524-local.tsv.gz contains a history of market
order books.  To understand what a limit order book is, see:
https://en.wikipedia.org/wiki/Order_book_(trading)
https://en.wikipedia.org/wiki/Order_(exchange)#Limit_order

trade-20190521-20190524-local.tsv.gz contains a history of actual trades
that took place.

To load the data, gunzip the files, and then in Python:

import pandas as pd

# load order book data
ob = pd.read_csv("order-books-20190521-20190524-local.tsv", sep="\t", parse_dates=["timestamp", "delivery_start", "delivery_end"])

# load trade data
tr = pd.read_csv("trade-20190521-20190524-local.tsv", sep="\t", parse_dates=["execution_time", "delivery_start", "delivery_end"])

Note the `sep="\t"`: the data is in TSV (tab-separated variable) format,
and some of the variables include whitespace, so we need to explicitly
specify tabs as the separator.

With this done you should have:

>>> ob.dtypes
line_number                       int64
idx                               int64
timestamp           datetime64[ns, UTC]
product                          object
contract_id                       int64
delivery_area_id                 object
long_name                        object
delivery_start      datetime64[ns, UTC]
delivery_end        datetime64[ns, UTC]
revision                          int64
last_px                           int64
last_qty                          int64
total_qty                         int64
best_ask_px                       int64
best_ask_qty                      int64
best_bid_px                       int64
best_bid_qty                      int64
dtype: object

>>> tr.dtypes
line_number                            int64
idx                                    int64
execution_time           datetime64[ns, UTC]
qty                                    int64
px                                     int64
product                               object
contract_id                            int64
buy_delivery_area_id                  object
sell_delivery_area_id                 object
long_name                             object
delivery_start           datetime64[ns, UTC]
delivery_end             datetime64[ns, UTC]
dtype: object

About the data fields
---------------------

`line_number` and `idx` can be ignored (although you could use `idx` as
your data index if you want something to match up against the text file).

`product` is the type of product being traded.  In this dataset there should
be only two types: `Intraday_Power_D` is used to trade electricity that will
be delivered over a 1-hour period, and `Quarterly_Hour_Power` is used to trade
electricity that will be delivered over a 15-minute period.

`contract_id` is the numerical identifier of an individual contract.  In this
data a contract is an instance of a product where the electricity has to be
supplied in a specific period of time (the "delivery period").  This means
that each contract has a 1:1 matching with some other data columns (described
in the next couple of paragraphs).

`long_name` is a string identifier for an individual contract, and consists
of the dates and times of the start and end of the delivery period, in Central
European Time.  There should be a 1:1 matching between this and `contract_id`.

`delivery_start` and `delivery_end` are UTC timestamps for the start and end
of the contract's delivery period.  There should be a 1:1 matching between the
pair of values and the `contract_id`.  Note that these values also match the
`long_name` once the timezone difference is taken into account.

For example:

contract_id  long_name                      delivery_start        delivery_end
11255534     20190522 02:45-20190522 03:00  2019-05-22T00:45:00Z  2019-05-22T01:00:00Z

`delivery_area_id` identifies the geographic region in which the electricity is
to be consumed or supplied.  Trading of contracts happens in multiple different
delivery areas, with one order book per (contract_id, delivery_area_id) pair.
This means that if we wanted to get only data for the order book for contract
11255534 in delivery area 10YDE-RWENET---I, we could do something like:

one_book = ob.loc[(ob["contract_id"] == 11255534) & (ob["delivery_area_id"] == "10YDE-RWENET---I")]

The trade data contains two delivery area fields: `buy_delivery_area_id` and
`sell_delivery_id` correspond to the delivery areas of the order books where
the buyer and seller placed their orders.  So if we wanted to get the series
of all trades of contract 11255534 that affected delivery area 10YDE-RWENET---I,
we could do:

one_trade = tr.loc[(tr["contract_id"] == 11255534) & ((tr["buy_delivery_area_id"] == "10YDE-RWENET---I")| (tr["sell_delivery_area_id"] == "10YDE-RWENET---I"))]

Trade between different delivery areas is only possible some of the time: see
if you can spot the pattern here.

There should be four different delivery areas in total, corresponding to four
different regions of Germany.

Order book data fields
----------------------

`timestamp` is the timestamp (in UTC) at which we received notification of a
change to the order book.

`best_bid_px` and `best_bid_qty` correspond to the limit price of the highest
priority BUY order in the market, and the quantity of power that order wants
to purchase.  ("Highest priority" means the highest buy price on offer.)

`best_ask_px` and `best_ask_qty` correspond to the limit price of the highest
priority SELL order in the market, and the quantity of power being offered.
("Highest priority" means the lowest sell price on offer.)

`last_px` and `last_qty` correspond to the price at which the last trade took
place on this order book, and the quantity that was traded.

`total_qty` corresponds to the total quantity traded overall via this limit
order book.

`revisionNo` can be ignored but is what it says -- a number that increases with
each individual change to the order book.

Trade data fields
-----------------

`execution_time` is the timestamp (in UTC) at which the trade took place, as
determined by the market maker.

`px` and `qty` are the price at which the trade took place and the quantity
of power that was traded.

Note that the trade data is much smaller than the order book data, because only
a subset of orders lead to trades.

About trading
-------------

The trading period for any individual contract should start about 1 hour before
the start of that contract's delivery period, and ends after 55 minutes (i.e.
about 5 minutes before the start of the delivery period).  Trading of that one
contract takes place in different order books for each delivery area.

At any given time the `best_ask_px` represents the lowest price at which someone
is willing to sell (hence the lowest price at which it is possible to buy), and
the `best_bid_px` represents the highest price someone is willing to pay (hence
the highest price at which it is possible to sell).

A successful trader needs to buy at lower prices and sell at higher prices (so
that they make money) but to end up with a ZERO quantity of power in each order
book at the end of the trading period, because we're assuming this trader is
just trading on the price fluctuations rather than having any actual ability to
supply or consume electricity.

Note that it's fine to sell when you currently have a zero quantity of power.
What you're selling in this case is a promise to supply that power during the
delivery period.  You just need to make sure you buy back that same quantity
before the trading period ends -- otherwise you will be expected to keep the
promise.

What to do with this data
-------------------------

Your challenge with this data is to come up with some data analysis or modelling
that might be of interest to someone trying to trade.  That might be coming up
with a simple trading strategy or learning mechanism, or identifying interesting
patterns in the data, or whatever else seems like it could be fun and useful.

We recommend writing your work up as a Python script or Jupyter Notebook, which
we can then talk through together in a follow-up discussion.

Feel free to ask us whatever questions you like, just as if you were a colleague
working remotely.  ESPECIALLY ask us questions if you get stuck understanding any
of the concepts in the data, or the meanings of any parameters, or if you run into
any technical difficulties using the data.

What we're looking for here is not a solution to a specific problem, but to get a
sense of what it would be like to work with you as a colleague -- the ways you
approach exploring data and problem-solving, and your ability to communicate your
ideas.  Have fun, be creative, and good luck!

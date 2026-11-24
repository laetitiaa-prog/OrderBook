# Orderbook

A C++20 limit order book and matching engine. Buy and sell orders are matched with
**price-time priority**, and the book supports the main order types found on real
exchanges.

## Features

- **Five order types**
  - `GoodTillCancel`: rests in the book until filled or cancelled
  - `GoodForDay`: automatically cancelled at 16:00 local time by a background thread
  - `FillAndKill`: matches what it can immediately, and the remainder is cancelled
  - `FillOrKill`: executes only if the *entire* quantity can be filled, otherwise rejected
  - `Market`: priced at the worst opposite level so it sweeps the book, then rests as GTC
- **Add, cancel and modify** orders (a modify is a cancel plus a re-add, which preserves correct time priority rules)
- **Trade reporting:** every match returns a `Trade` with both sides' order ID, price and quantity
- **Book snapshots:** `GetOrderInfos()` returns aggregated quantity per price level for bids and asks
- **Data-driven tests:** scenarios are plain text files, so new test cases need no new C++

## How it works

| Concern | Approach |
|---------|----------|
| Price levels | `std::map` per side, ordered so the best bid/ask is always `begin()` |
| Time priority | Each level is a FIFO `std::list` of orders |
| Cancel lookup | `unordered_map<OrderId, {order, list iterator}>` gives O(1) lookup and removal from a level |
| Fill-Or-Kill check | Per-price aggregate (`quantity`, `count`) is walked to check whether the full size is fillable |
| Good-For-Day expiry | A worker thread sleeps on a `condition_variable` until 16:00, then cancels all GFD orders; it shuts down cleanly in the destructor |

## Getting started

**Requirements:** Windows, Visual Studio 2022+ (MSVC, C++20), NuGet

1. Open `Orderbook.slnx` in Visual Studio.
2. Build the solution (NuGet restores GoogleTest on first build).
3. Run the tests from **Test > Test Explorer**.

## Testing

Tests use [GoogleTest](https://github.com/google/googletest) and are driven by the
scenario files in `TestFiles/`. Each file lists actions, then the expected final book state:

```
A B GoodTillCancel 100 10 1   # Add: side, type, price, quantity, orderId
M 1 B 101 5                   # Modify: orderId, side, price, quantity
C 1                           # Cancel: orderId
R 0 0 0                       # Result: total orders, bid levels, ask levels
```

Scenarios cover GTC matching, Fill-And-Kill, Fill-Or-Kill (hit and miss),
cancellation, modifying an order's side, and market orders.

## License

MIT
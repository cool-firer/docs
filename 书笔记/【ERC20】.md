# 两个关键方法

## approve

```javascript
function approve(address spender, uint256 value) public virtual returns (bool) {
  address owner = _msgSender();
  _approve(owner, spender, value);
  return true;
}
```

语义：I give spender quota，即，我给spender一张面额value的支票。并不是真实的钱。那spender要如何兑现这张支票呢？

## transferFrom

```javascript
function transferFrom(address from, address to, uint256 value) public virtual returns (bool) {
  address spender = _msgSender();
  _spendAllowance(from, spender, value);
  _transfer(from, to, value);
  return true;
}
```

语义：I wanna use the quota from 【from】，transfer to 【to】，即，我要兑现from给我的支票，把支票上value这么多的钱，转到to账户上。这将会发起真正的钱转账。
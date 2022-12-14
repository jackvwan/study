# non-capture group And look-arounds

## non-capture group

> "?:" 是非俘获组，不会被俘获。会被当作匹配字符串。

如：

```regex
abcd1234abcd

regex: (?:[a-z]*)1234
```

将会匹配到 abcd1234

## look-arounds

look-arounds 只起到定位的作用，不会被当作匹配字符串，因此，只能用于整个模式的两端。

> "?=" 肯定先行预测。在前面的模式之后需要有此表达式。
>
> "?!" 否定先行预测。在前面的模式之后不能有此表达式。
>
> "?<=" 肯定。在后续的模式之前需要有此表达式。
>
> "?<!" 反向否定断言。在后续的模式之前不能有此表达式。

如

```regex
abcd1234abcd

regex: (?=[a-z]*)1234
```

将会匹配到 1234，前面由**肯定先行预测**表示的abcd不会被包含

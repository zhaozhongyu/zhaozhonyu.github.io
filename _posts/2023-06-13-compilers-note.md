---
layout: post
title: 编译原理笔记-词法分析(tokenizer)
subtitle: 编译原理笔记-词法分析(tokenizer)-1
tags: [compilers]
comments: true
---

## 词法分析的过程，其实就是对一个字符串进行模式匹配的过程.
在词法分析的过程中, 它是上下文无关的一个操作. 根据我们对于词法分析的原理学习, 已经知道词法分析的本质就是根据字符串的不同, 进行匹配不同的模式, 根据不同模式得到不同的token, 最后得到一个token的列表.

## 源码
词法分析的核心, 毫无疑问就是读出token的函数

#### `readToken`

```javascript
/**
 * 读出一个token
 * @param {number} code 
 * @returns {Token}
 */
pp.readToken = function(code) {
  // Identifier or keyword. '\uXXXX' sequences are allowed in
  // identifiers, so '\' also dispatches to that.
  if (isIdentifierStart(code, this.options.ecmaVersion >= 6) || code === 92 /* '\' */)
    return this.readWord()

  return this.getTokenFromCode(code)
}
```
从代码中可以看出, 这里分2种情况, 一种是如果字符串以下划线或者英文字符开头, 则此时调用`readWord`, 否则调用`getTokenFromCode(code)`作为返回.

#### `getTokenFromCode(code)`
根据第一个字符的, 使用switch方法调用不同的方法进行处理, 主要分2种处理类型: 
1. finishToken, 当前的字符已经是结束符, 又或者这个字符本身就作为一个独立的分隔符或者是作用域等.
2. 根据当前的字符类型, 继续往下读内容, 常见的操作类型如下:
    1. 读到0, 此时往下读不同进制的数字, 如0x, 0o, 0b
    2. +/-, 1-9, 此时往下读数字
    3. `'`, `"`此时往下读字符串
    4. 其他操作符, 如`*`,`/`, `%`, 三元操作符`?`, 以及逻辑操作符`|`, `&`, 
4. Default 抛出错误.

```javascript
/**
 * 根据传入的code值, 区分获取不一样的token, 进入这个方法说明不是字母, 下划线
 * @param {number} code 
 * @returns {Token}
 */
pp.getTokenFromCode = function(code) {
  switch (code) {
  // The interpretation of a dot depends on whether it is followed
  // by a digit or another two dots.
  case 46: // '.'
    return this.readToken_dot()

  // Punctuation tokens.
  case 40: ++this.pos; return this.finishToken(tt.parenL) // (
  case 41: ++this.pos; return this.finishToken(tt.parenR) // )
  case 59: ++this.pos; return this.finishToken(tt.semi) // ;
  case 44: ++this.pos; return this.finishToken(tt.comma) // ,
  case 91: ++this.pos; return this.finishToken(tt.bracketL) // [
  case 93: ++this.pos; return this.finishToken(tt.bracketR) // ]
  case 123: ++this.pos; return this.finishToken(tt.braceL) // {
  case 125: ++this.pos; return this.finishToken(tt.braceR) // }
  case 58: ++this.pos; return this.finishToken(tt.colon) // :

  case 96: // '`'
    if (this.options.ecmaVersion < 6) break
    ++this.pos
    return this.finishToken(tt.backQuote)

  case 48: // '0'
    let next = this.input.charCodeAt(this.pos + 1)
    if (next === 120 || next === 88) return this.readRadixNumber(16) // '0x', '0X' - hex number
    if (this.options.ecmaVersion >= 6) {
      if (next === 111 || next === 79) return this.readRadixNumber(8) // '0o', '0O' - octal number
      if (next === 98 || next === 66) return this.readRadixNumber(2) // '0b', '0B' - binary number
    }

  // Anything else beginning with a digit is an integer, octal
  // number, or float.
  case 49: case 50: case 51: case 52: case 53: case 54: case 55: case 56: case 57: // 1-9
    return this.readNumber(false)

  // Quotes produce strings.
  case 34: case 39: // '"', "'"
    return this.readString(code)

  // Operators are parsed inline in tiny state machines. '=' (61) is
  // often referred to. `finishOp` simply skips the amount of
  // characters it is given as second argument, and returns a token
  // of the type given by its first argument.
  case 47: // '/'
    return this.readToken_slash()

  case 37: case 42: // '%*'
    return this.readToken_mult_modulo_exp(code)

  case 124: case 38: // '|&'
    return this.readToken_pipe_amp(code)

  case 94: // '^'
    return this.readToken_caret()

  case 43: case 45: // '+-'
    return this.readToken_plus_min(code)

  case 60: case 62: // '<>'
    return this.readToken_lt_gt(code)

  case 61: case 33: // '=!'
    return this.readToken_eq_excl(code)

  case 63: // '?'
    return this.readToken_question()

  case 126: // '~'
    return this.finishOp(tt.prefix, 1)

  case 35: // '#'
    return this.readToken_numberSign()
  }

  this.raise(this.pos, "Unexpected character '" + codePointToString(code) + "'")
}
```

### 首先看数字读取
#### `readInt`
在readInt中主要做的操作是根据给的radix, 往下读出数字, 直到遇到一个非数字字符, 然后返回读到的数字(十进制). 这个方法是读取多进制和读取数字时的方法.
```javascript
// Read an integer in the given radix. Return null if zero digits
// were read, the integer value otherwise. When `len` is given, this
// will return `null` unless the integer has exactly `len` digits.
/**
 * 
 * @param {number} radix 
 * @param {number | undefined} len 
 * @param {boolean} maybeLegacyOctalNumericLiteral 
 * @returns 
 */
pp.readInt = function(radix, len, maybeLegacyOctalNumericLiteral) {
  // `len` is used for character escape sequences. In that case, disallow separators.
  const allowSeparators = this.options.ecmaVersion >= 12 && len === undefined

  // `maybeLegacyOctalNumericLiteral` is true if it doesn't have prefix (0x,0o,0b)
  // and isn't fraction part nor exponent part. In that case, if the first digit
  // is zero then disallow separators.
  const isLegacyOctalNumericLiteral = maybeLegacyOctalNumericLiteral && this.input.charCodeAt(this.pos) === 48

  let start = this.pos, total = 0, lastCode = 0
  for (let i = 0, e = len == null ? Infinity : len; i < e; ++i, ++this.pos) {
    let code = this.input.charCodeAt(this.pos), val

    if (allowSeparators && code === 95) {
      if (isLegacyOctalNumericLiteral) this.raiseRecoverable(this.pos, "Numeric separator is not allowed in legacy octal numeric literals")
      if (lastCode === 95) this.raiseRecoverable(this.pos, "Numeric separator must be exactly one underscore")
      if (i === 0) this.raiseRecoverable(this.pos, "Numeric separator is not allowed at the first of digits")
      lastCode = code
      continue
    }

    if (code >= 97) val = code - 97 + 10 // a
    else if (code >= 65) val = code - 65 + 10 // A
    else if (code >= 48 && code <= 57) val = code - 48 // 0-9
    else val = Infinity
    if (val >= radix) break
    lastCode = code
    total = total * radix + val
  }

  if (allowSeparators && lastCode === 95) this.raiseRecoverable(this.pos - 1, "Numeric separator is not allowed at the last of digits")
  if (this.pos === start || len != null && this.pos - start !== len) return null

  return total
}
```

#### `readRadixNumber`
在读取出数字后, 还做了一个判断, 如果数字后接了字母n, 也就是`1234n`格式时, 此时js的语义里, 此数字为bigint, 此时抛弃readInt读到的数字, 使用`stringToBigInt`进行转换.最后完成为num类型.
```javascript
/**
 * 读出多进制的数字, 主要是2, 8, 16
 */
pp.readRadixNumber = function(radix) {
  let start = this.pos
  this.pos += 2 // 0x
  let val = this.readInt(radix)
  if (val == null) this.raise(this.start + 2, "Expected number in radix " + radix)
  if (this.options.ecmaVersion >= 11 && this.input.charCodeAt(this.pos) === 110) {
    val = stringToBigInt(this.input.slice(start, this.pos))
    ++this.pos
  } else if (isIdentifierStart(this.fullCharCodeAtPos())) this.raise(this.pos, "Identifier directly after number")
  return this.finishToken(tt.num, val)
}
```

#### `readNumber`
readNumber有一个参数, 如果是前面读到了`.`号的时候, 则此时会做一个判断是否后面有接参数. 
在往下读的过程中, 会分别判断`.`, `n`, `e`的存在, 并校验对应的语法.
但获取数字的核心是`stringToNumber`函数, 该函数直接调用了`parseFloat`进行解析返回(朴实无华).

```javascript
/**
 * 读出10进制数字
 * @param {boolean} startsWithDot
 */
pp.readNumber = function(startsWithDot) {
  let start = this.pos
  if (!startsWithDot && this.readInt(10, undefined, true) === null) this.raise(start, "Invalid number")
  let octal = this.pos - start >= 2 && this.input.charCodeAt(start) === 48
  if (octal && this.strict) this.raise(start, "Invalid number")
  let next = this.input.charCodeAt(this.pos)
  // 前面调用了一次readInt, 所以这里pos已经移动到下一个位置了
  if (!octal && !startsWithDot && this.options.ecmaVersion >= 11 && next === 110) {
    let val = stringToBigInt(this.input.slice(start, this.pos))
    ++this.pos
    if (isIdentifierStart(this.fullCharCodeAtPos())) this.raise(this.pos, "Identifier directly after number")
    return this.finishToken(tt.num, val)
  }
  if (octal && /[89]/.test(this.input.slice(start, this.pos))) octal = false
  if (next === 46 && !octal) { // '.'
    ++this.pos
    this.readInt(10)
    next = this.input.charCodeAt(this.pos)
  }
  if ((next === 69 || next === 101) && !octal) { // 'eE'
    next = this.input.charCodeAt(++this.pos)
    if (next === 43 || next === 45) ++this.pos // '+-'
    if (this.readInt(10) === null) this.raise(start, "Invalid number")
  }
  if (isIdentifierStart(this.fullCharCodeAtPos())) this.raise(this.pos, "Identifier directly after number")

  let val = stringToNumber(this.input.slice(start, this.pos), octal)
  return this.finishToken(tt.num, val)
}
```
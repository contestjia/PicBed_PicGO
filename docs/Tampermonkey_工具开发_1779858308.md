---
title: Tampermonkey 工具开发
---

# Tampermonkey 工具开发

#### ChatGPT:
可以。Tampermonkey 脚本里也按同样方式改，关键区别是：**Tampermonkey 版本用了 Shadow DOM，所以自动调整高度要用 `root.querySelectorAll(...)`，不能用 `document.querySelectorAll(...)`。**

把你的 Tampermonkey 脚本按下面几处修改。

## 1. 替换 `textarea` CSS

找到脚本里的：

```css
textarea {
  width: 100%;
  resize: vertical;
  border: 1px solid #3578e5;
  border-radius: 4px;
  background: #eee;
  color: #333;
  padding: 7px 8px;
  font: 18px/1.35 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  white-space: pre;
}
```

替换成：

```css
textarea {
  width: 100%;
  resize: vertical;
  overflow: hidden;
  border: 1px solid #3578e5;
  border-radius: 4px;
  background: #eee;
  color: #333;
  padding: 7px 8px;
  font: 18px/1.35 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  word-break: break-word;
}
```

---

## 2. 修改 `makeValueRow`

找到：

```javascript
function makeValueRow(label, value) {
  return `
    <label class="value-row">
      <span>${escapeHtml(label)}:</span>
      <textarea readonly rows="${getRows(value)}">${escapeHtml(value)}</textarea>
    </label>
  `;
}
```

替换成：

```javascript
function makeValueRow(label, value) {
  return `
    <label class="value-row">
      <span>${escapeHtml(label)}:</span>
      <textarea readonly wrap="soft" rows="${getRows(value)}">${escapeHtml(value)}</textarea>
    </label>
  `;
}
```

---

## 3. 在 `createUI()` 里面新增自动调整高度函数

在 `createUI()` 里面，找到这几行：

```javascript
const panel = $('.panel');
const input = $('#fp-decimal');
const result = $('.result');
```

下面加上：

```javascript
function autoResizeTextareas() {
  root.querySelectorAll('.result textarea').forEach((textarea) => {
    textarea.style.height = 'auto';
    textarea.style.height = `${textarea.scrollHeight + 2}px`;
  });
}
```

注意这里用的是：

```javascript
root.querySelectorAll(...)
```

不要写成：

```javascript
document.querySelectorAll(...)
```

因为你的 Tampermonkey UI 是挂在 Shadow DOM 里的。

---

## 4. 在 `convert()` 里调用自动调整

找到 `convert()` 里面这一行：

```javascript
result.innerHTML = buildResultHtml(parsed, results, opts);
```

替换成：

```javascript
result.innerHTML = buildResultHtml(parsed, results, opts);
autoResizeTextareas();
```

---

改完后，长输出内容会在 textarea 内自动换行，并且输出框高度会根据内容自动撑开，不会再横向显示很长一行。



#### You:
直接给我最终的可复制的脚本就行

#### ChatGPT:
下面是最终可复制版：**无右下角 FP 按钮、默认隐藏、按 Alt + 4 显示 / 隐藏、输出框自动换行并自动撑高**。

```javascript
// ==UserScript==
// @name         Decimal to Floating-Point Converter
// @namespace    https://chatgpt.com/
// @version      1.2.0
// @description  Decimal to IEEE-754 float32 / float64 converter, Alt+4 toggle, auto-wrap output
// @author       ChatGPT
// @match        http://*/*
// @match        https://*/*
// @run-at       document-end
// @grant        none
// ==/UserScript==

(() => {
  'use strict';

  const FP32 = {
    name: 'Single',
    label: 'Single / float32',
    expBits: 8,
    fracBits: 23,
    bias: 127,
    totalBits: 32,
  };

  const FP64 = {
    name: 'Double',
    label: 'Double / float64',
    expBits: 11,
    fracBits: 52,
    bias: 1023,
    totalBits: 64,
  };

  function bitLength(n) {
    return n === 0n ? 0 : n.toString(2).length;
  }

  function pow10(n) {
    return 10n ** BigInt(n);
  }

  function parseDecimalExact(str) {
    const s = str.trim();
    const lower = s.toLowerCase();

    if (lower === 'nan') {
      return { special: 'nan', sign: 0, raw: s };
    }

    if (
      lower === 'infinity' ||
      lower === '+infinity' ||
      lower === 'inf' ||
      lower === '+inf'
    ) {
      return { special: 'inf', sign: 0, raw: s };
    }

    if (lower === '-infinity' || lower === '-inf') {
      return { special: 'inf', sign: 1, raw: s };
    }

    const m = s.match(
      /^([+-])?(?:(\d+)(?:\.(\d*))?|\.(\d+))(?:[eE]([+-]?\d+))?$/
    );

    if (!m) {
      throw new Error('请输入十进制数，例如 0.9、999999.9、3.1415、1.56e-11、4e20。');
    }

    const sign = m[1] === '-' ? 1 : 0;
    const intPart = m[2] ?? '0';
    const fracPart = m[4] !== undefined ? m[4] : (m[3] ?? '');
    const exp10 = Number(m[5] ?? '0');

    if (!Number.isSafeInteger(exp10) || Math.abs(exp10) > 20000) {
      throw new Error('指数太大，可能导致浏览器卡顿；请缩小输入范围。');
    }

    let digits = (intPart + fracPart).replace(/^0+/, '') || '0';

    if (digits.length + Math.abs(exp10) > 20000) {
      throw new Error('输入太大，可能导致浏览器卡顿；请缩小输入范围。');
    }

    let numerator = BigInt(digits);
    let denominator = 1n;
    const scale = fracPart.length - exp10;

    if (scale > 0) {
      denominator = pow10(scale);
    } else if (scale < 0) {
      numerator *= pow10(-scale);
    }

    return {
      special: null,
      sign,
      numerator,
      denominator,
      raw: s,
    };
  }

  function cmpToPow2(n, d, e) {
    let a;
    let b;

    if (e >= 0) {
      a = n;
      b = d << BigInt(e);
    } else {
      a = n << BigInt(-e);
      b = d;
    }

    if (a > b) return 1;
    if (a < b) return -1;
    return 0;
  }

  function floorLog2Rational(n, d) {
    let e = bitLength(n) - bitLength(d);

    while (cmpToPow2(n, d, e) < 0) e--;
    while (cmpToPow2(n, d, e + 1) >= 0) e++;

    return e;
  }

  function roundScaled(n, d, shift) {
    let a;
    let b;

    if (shift >= 0) {
      a = n << BigInt(shift);
      b = d;
    } else {
      a = n;
      b = d << BigInt(-shift);
    }

    let q = a / b;
    const r = a % b;
    const twice = r * 2n;

    if (twice > b || (twice === b && (q & 1n) === 1n)) {
      q++;
    }

    return q;
  }

  function convertToIEEE(parsed, fmt) {
    const { expBits, fracBits, bias } = fmt;
    const maxExpField = (1 << expBits) - 1;

    if (parsed.special === 'nan') {
      return {
        fmt,
        sign: 0,
        expField: maxExpField,
        frac: 1n << BigInt(fracBits - 1),
        special: 'nan',
        subnormal: false,
      };
    }

    if (parsed.special === 'inf') {
      return {
        fmt,
        sign: parsed.sign,
        expField: maxExpField,
        frac: 0n,
        special: 'inf',
        subnormal: false,
      };
    }

    const n = parsed.numerator;
    const d = parsed.denominator;

    if (n === 0n) {
      return {
        fmt,
        sign: parsed.sign,
        expField: 0,
        frac: 0n,
        special: null,
        subnormal: false,
      };
    }

    const minE = 1 - bias;
    const maxE = bias;
    const hidden = 1n << BigInt(fracBits);

    let e = floorLog2Rational(n, d);
    let expField;
    let frac;

    if (e < minE) {
      const q = roundScaled(n, d, fracBits - minE);

      if (q === 0n) {
        expField = 0;
        frac = 0n;
      } else if (q >= hidden) {
        expField = 1;
        frac = 0n;
      } else {
        expField = 0;
        frac = q;
      }
    } else {
      let q = roundScaled(n, d, fracBits - e);

      if (q >= (hidden << 1n)) {
        q >>= 1n;
        e++;
      }

      if (e > maxE) {
        expField = maxExpField;
        frac = 0n;
      } else {
        expField = e + bias;
        frac = q - hidden;
      }
    }

    return {
      fmt,
      sign: parsed.sign,
      expField,
      frac,
      special: null,
      subnormal: expField === 0 && frac !== 0n,
    };
  }

  function rawBitsBigInt(r) {
    return (
      (BigInt(r.sign) << BigInt(r.fmt.expBits + r.fmt.fracBits)) |
      (BigInt(r.expField) << BigInt(r.fmt.fracBits)) |
      r.frac
    );
  }

  function rawBinary(r) {
    return `${r.sign} ${r.expField.toString(2).padStart(r.fmt.expBits, '0')} ${r.frac
      .toString(2)
      .padStart(r.fmt.fracBits, '0')}`;
  }

  function rawHex(r) {
    return '0x' + rawBitsBigInt(r).toString(16).padStart(r.fmt.totalBits / 4, '0');
  }

  function finiteParts(r) {
    const fracBits = r.fmt.fracBits;
    const bias = r.fmt.bias;
    const maxExpField = (1 << r.fmt.expBits) - 1;

    if (r.expField === maxExpField) return null;

    if (r.expField === 0) {
      return {
        sign: r.sign,
        sig: r.frac,
        exp2: (1 - bias) - fracBits,
      };
    }

    return {
      sign: r.sign,
      sig: (1n << BigInt(fracBits)) + r.frac,
      exp2: (r.expField - bias) - fracBits,
    };
  }

  function exactDecimalString(r) {
    const maxExpField = (1 << r.fmt.expBits) - 1;

    if (r.expField === maxExpField) {
      return r.frac === 0n ? (r.sign ? '-Infinity' : 'Infinity') : 'NaN';
    }

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return p.sign ? '-0' : '0';
    }

    let out;

    if (p.exp2 >= 0) {
      out = (p.sig << BigInt(p.exp2)).toString();
    } else {
      const k = -p.exp2;
      let digits = (p.sig * (5n ** BigInt(k))).toString();

      if (digits.length <= k) {
        digits = '0'.repeat(k - digits.length + 1) + digits;
      }

      const idx = digits.length - k;
      out = `${digits.slice(0, idx)}.${digits.slice(idx)}`;
      out = out.replace(/\.0+$/, '').replace(/(\.\d*?)0+$/, '$1');
    }

    return p.sign ? '-' + out : out;
  }

  function exactBinaryString(r) {
    const maxExpField = (1 << r.fmt.expBits) - 1;

    if (r.expField === maxExpField) {
      return r.frac === 0n ? (r.sign ? '-Infinity' : 'Infinity') : 'NaN';
    }

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return p.sign ? '-0' : '0';
    }

    let out;

    if (p.exp2 >= 0) {
      out = (p.sig << BigInt(p.exp2)).toString(2);
    } else {
      const k = -p.exp2;
      const bits = p.sig.toString(2);

      if (bits.length <= k) {
        out = '0.' + '0'.repeat(k - bits.length) + bits;
      } else {
        out = `${bits.slice(0, bits.length - k)}.${bits.slice(bits.length - k)}`;
      }

      out = out.replace(/\.0+$/, '').replace(/(\.\d*?)0+$/, '$1');
    }

    return p.sign ? '-' + out : out;
  }

  function rationalEqualToResult(parsed, r) {
    if (parsed.special) {
      return parsed.special === r.special;
    }

    const maxExpField = (1 << r.fmt.expBits) - 1;

    if (r.expField === maxExpField) {
      return false;
    }

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return parsed.numerator === 0n;
    }

    let rn = p.sig;
    let rd = 1n;

    if (p.exp2 >= 0) {
      rn <<= BigInt(p.exp2);
    } else {
      rd <<= BigInt(-p.exp2);
    }

    return parsed.numerator * rd === rn * parsed.denominator;
  }

  function formatPow(base, exp) {
    return ` * ${base}^${exp}`;
  }

  function specialValueString(r) {
    const maxExpField = (1 << r.fmt.expBits) - 1;

    if (r.expField !== maxExpField) return null;

    if (r.frac === 0n) return r.sign ? '-Infinity' : 'Infinity';
    return 'NaN';
  }

  function withSign(sign, text) {
    if (!sign) return text;
    if (text === 'NaN') return text;
    return '-' + text;
  }

  function decimalFromIntegerTimesPowerOfTwo(sig, exp2) {
    if (sig === 0n) return '0';

    if (exp2 >= 0) {
      return (sig << BigInt(exp2)).toString();
    }

    const k = -exp2;
    let digits = (sig * (5n ** BigInt(k))).toString();

    if (digits.length <= k) {
      digits = '0'.repeat(k - digits.length + 1) + digits;
    }

    const idx = digits.length - k;
    let out = `${digits.slice(0, idx)}.${digits.slice(idx)}`;

    out = out.replace(/\.0+$/, '').replace(/(\.\d*?)0+$/, '$1');
    return out;
  }

  function normalizedDecimalScientific(r) {
    const special = specialValueString(r);
    if (special) return special;

    let dec = exactDecimalString(r);
    let sign = '';

    if (dec.startsWith('-')) {
      sign = '-';
      dec = dec.slice(1);
    }

    if (dec === '0' || dec === '-0') return sign + '0';

    let intPart = dec;
    let fracPart = '';

    if (dec.includes('.')) {
      [intPart, fracPart] = dec.split('.');
    }

    let exponent;
    let digits;

    if (/[1-9]/.test(intPart)) {
      const cleanInt = intPart.replace(/^0+/, '');
      exponent = cleanInt.length - 1;
      digits = cleanInt + fracPart;
    } else {
      const firstNonZero = fracPart.search(/[1-9]/);

      if (firstNonZero < 0) return sign + '0';

      exponent = -(firstNonZero + 1);
      digits = fracPart.slice(firstNonZero);
    }

    digits = digits.replace(/^0+/, '').replace(/0+$/, '');

    if (!digits) return sign + '0';

    const mantissa = digits.length === 1 ? digits : `${digits[0]}.${digits.slice(1)}`;
    return `${sign}${mantissa}${formatPow(10, exponent)}`;
  }

  function normalizedBinaryScientific(r) {
    const special = specialValueString(r);
    if (special) return special;

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return p.sign ? '-0' : '0';
    }

    const bits = p.sig.toString(2);
    const exponent = p.exp2 + bits.length - 1;
    const frac = bits.slice(1).replace(/0+$/, '');
    const mantissa = frac ? `1.${frac}` : '1';

    return `${p.sign ? '-' : ''}${mantissa}${formatPow(2, exponent)}`;
  }

  function normalizedDecimalTimesPow2(r) {
    const special = specialValueString(r);
    if (special) return special;

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return p.sign ? '-0 * 2^0' : '0 * 2^0';
    }

    const bits = p.sig.toString(2);
    const exponent = p.exp2 + bits.length - 1;

    const mantissa = decimalFromIntegerTimesPowerOfTwo(
      p.sig,
      -(bits.length - 1)
    );

    return `${p.sign ? '-' : ''}${mantissa}${formatPow(2, exponent)}`;
  }

  function decimalIntegerTimesPow2(r) {
    const special = specialValueString(r);
    if (special) return special;

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return p.sign ? '-0 * 2^0' : '0 * 2^0';
    }

    return `${p.sign ? '-' : ''}${p.sig.toString()}${formatPow(2, p.exp2)}`;
  }

  function decimalIntegerTimesPow10(r) {
    const special = specialValueString(r);
    if (special) return special;

    let dec = exactDecimalString(r);
    let sign = '';

    if (dec.startsWith('-')) {
      sign = '-';
      dec = dec.slice(1);
    }

    if (dec === '0' || dec === '-0') {
      return sign + '0 * 10^0';
    }

    let exp10 = 0;
    let digits = dec;

    if (dec.includes('.')) {
      const [intPart, fracPart] = dec.split('.');
      digits = intPart + fracPart;
      exp10 = -fracPart.length;
    }

    digits = digits.replace(/^0+/, '');

    if (!digits) {
      return sign + '0 * 10^0';
    }

    const trailingZeros = digits.match(/0+$/)?.[0]?.length ?? 0;

    if (trailingZeros > 0) {
      digits = digits.slice(0, -trailingZeros);
      exp10 += trailingZeros;
    }

    return `${sign}${digits}${formatPow(10, exp10)}`;
  }

  function hexadecimalFloatingPointConstant(r) {
    const special = specialValueString(r);
    if (special) return special;

    const p = finiteParts(r);

    if (p.sig === 0n) {
      return p.sign ? '-0x0p0' : '0x0p0';
    }

    const bits = p.sig.toString(2);
    const exponent = p.exp2 + bits.length - 1;

    let fracBits = bits.slice(1);

    if (fracBits.length > 0) {
      const pad = (4 - (fracBits.length % 4)) % 4;
      fracBits += '0'.repeat(pad);
    }

    let fracHex = '';

    if (fracBits.length > 0) {
      fracHex = BigInt('0b' + fracBits)
        .toString(16)
        .padStart(fracBits.length / 4, '0')
        .replace(/0+$/, '');
    }

    const body = fracHex ? `0x1.${fracHex}p${exponent}` : `0x1p${exponent}`;
    return withSign(p.sign, body);
  }

  function escapeHtml(s) {
    return String(s)
      .replaceAll('&', '&amp;')
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;');
  }

  function getRows(text) {
    const len = String(text).length;

    if (len > 300) return 5;
    if (len > 130) return 3;
    return 2;
  }

  function makeResultBlock(title, rows) {
    return `
      <div class="result-block">
        <h3>${escapeHtml(title)}</h3>
        ${rows.join('')}
      </div>
    `;
  }

  function makeValueRow(label, value) {
    return `
      <label class="value-row">
        <span>${escapeHtml(label)}:</span>
        <textarea readonly wrap="soft" rows="${getRows(value)}">${escapeHtml(value)}</textarea>
      </label>
    `;
  }

  function buildResultHtml(parsed, results, options) {
    const selected = results.filter(Boolean);
    const blocks = [];

    if (options.decimal) {
      blocks.push(
        makeResultBlock(
          'Decimal',
          selected.map((r) => makeValueRow(r.fmt.name, exactDecimalString(r)))
        )
      );
    }

    if (options.binary) {
      blocks.push(
        makeResultBlock(
          'Binary',
          selected.map((r) => makeValueRow(r.fmt.name, exactBinaryString(r)))
        )
      );
    }

    if (options.normDecimalScientific) {
      blocks.push(
        makeResultBlock(
          'Normalized decimal scientific notation',
          selected.map((r) => makeValueRow(r.fmt.name, normalizedDecimalScientific(r)))
        )
      );
    }

    if (options.normBinaryScientific) {
      blocks.push(
        makeResultBlock(
          'Normalized binary scientific notation',
          selected.map((r) => makeValueRow(r.fmt.name, normalizedBinaryScientific(r)))
        )
      );
    }

    if (options.normDecimalTimesPow2) {
      blocks.push(
        makeResultBlock(
          'Normalized decimal times a power of two',
          selected.map((r) => makeValueRow(r.fmt.name, normalizedDecimalTimesPow2(r)))
        )
      );
    }

    if (options.intTimesPow2) {
      blocks.push(
        makeResultBlock(
          'Decimal integer times a power of two',
          selected.map((r) => makeValueRow(r.fmt.name, decimalIntegerTimesPow2(r)))
        )
      );
    }

    if (options.intTimesPow10) {
      blocks.push(
        makeResultBlock(
          'Decimal integer times a power of ten',
          selected.map((r) => makeValueRow(r.fmt.name, decimalIntegerTimesPow10(r)))
        )
      );
    }

    if (options.hexFloat) {
      blocks.push(
        makeResultBlock(
          'Hexadecimal floating-point constant',
          selected.map((r) => makeValueRow(r.fmt.name, hexadecimalFloatingPointConstant(r)))
        )
      );
    }

    if (options.rawBinary) {
      blocks.push(
        makeResultBlock(
          'Raw Binary (sign field | exponent field | significand field)',
          selected.map((r) => makeValueRow(r.fmt.name, rawBinary(r)))
        )
      );
    }

    if (options.rawHex) {
      blocks.push(
        makeResultBlock(
          'Raw Hexadecimal',
          selected.map((r) => makeValueRow(r.fmt.name, rawHex(r)))
        )
      );
    }

    if (options.flags) {
      const rows = selected.map((r) => {
        const inexact = !rationalEqualToResult(parsed, r);
        const subnormal = r.subnormal === true;

        return `
          <div class="flag-row">
            <b>${escapeHtml(r.fmt.name)}:</b>
            <span class="flag ${inexact ? 'on' : ''}">${inexact ? '☑' : '☐'} Inexact</span>
            <span class="flag ${subnormal ? 'on' : ''}">${subnormal ? '☑' : '☐'} Subnormal</span>
          </div>
        `;
      });

      blocks.push(makeResultBlock('Flags', rows));
    }

    return `
      <section class="out">
        <h2>Floating-Point</h2>
        <p>Converts to this binary floating-point number:</p>
        ${blocks.join('')}
      </section>
    `;
  }

  function createUI() {
    if (document.getElementById('__fp_converter_tm_host__')) return;

    const host = document.createElement('div');
    host.id = '__fp_converter_tm_host__';
    document.documentElement.appendChild(host);

    const root = host.attachShadow({ mode: 'open' });

    root.innerHTML = `
      <style>
        :host {
          all: initial;
        }

        *, *::before, *::after {
          box-sizing: border-box;
        }

        .panel {
          position: fixed;
          z-index: 2147483647;
          right: 22px;
          top: 22px;
          width: min(980px, calc(100vw - 44px));
          max-height: min(900px, calc(100vh - 44px));
          overflow: auto;
          background: #fff;
          color: #111;
          border: 1px solid #b7c7e8;
          border-radius: 12px;
          box-shadow: 0 12px 40px rgba(0,0,0,.28);
          font: 15px/1.45 Georgia, "Times New Roman", serif;
          display: block;
        }

        .panel.closed {
          display: none;
        }

        .top {
          position: sticky;
          top: 0;
          z-index: 1;
          display: flex;
          align-items: center;
          justify-content: space-between;
          gap: 12px;
          padding: 14px 18px;
          border-bottom: 1px solid #dbe5ff;
          background: #fff;
        }

        .top h1 {
          margin: 0;
          font: 800 28px/1.15 system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
          letter-spacing: .2px;
        }

        .shortcut {
          color: #666;
          font: 13px/1.3 system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
          white-space: nowrap;
        }

        .close {
          border: 1px solid #aaa;
          background: #f7f7f7;
          border-radius: 8px;
          font: 700 18px/1 system-ui, sans-serif;
          width: 34px;
          height: 34px;
          cursor: pointer;
        }

        .body {
          padding: 16px 18px 20px;
        }

        h2 {
          margin: 0 0 6px;
          color: #088000;
          font-size: 25px;
        }

        .out h2 {
          color: #3b7bdc;
        }

        h3 {
          margin: 16px 0 8px;
          font-size: 17px;
          font-style: italic;
        }

        p {
          margin: 0 0 10px;
        }

        .input-line {
          display: grid;
          gap: 6px;
        }

        input[type="text"] {
          width: 100%;
          border: 1px solid #10a025;
          border-radius: 4px;
          padding: 9px 10px;
          font: 20px/1.2 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
        }

        button.action {
          margin: 8px 8px 0 0;
          padding: 5px 12px;
          border: 1px solid #999;
          border-radius: 8px;
          background: #fff;
          font: 800 18px/1.1 system-ui, sans-serif;
          cursor: pointer;
        }

        .opts {
          margin-top: 16px;
          display: grid;
          gap: 5px;
        }

        .opts-title {
          font-weight: 700;
          margin-bottom: 2px;
        }

        .opt-row {
          display: flex;
          flex-wrap: wrap;
          align-items: center;
          gap: 14px;
        }

        .fmt-grid {
          display: grid;
          grid-template-columns: repeat(2, minmax(320px, 1fr));
          gap: 4px 18px;
          max-width: 900px;
          margin-left: 14px;
        }

        label {
          cursor: pointer;
        }

        input[type="checkbox"] {
          transform: translateY(1px);
          margin-right: 4px;
        }

        .out {
          margin-top: 18px;
          padding-top: 10px;
          border-top: 5px solid #d7e7ff;
        }

        .value-row {
          display: grid;
          gap: 4px;
          margin: 8px 0;
          cursor: default;
        }

        .value-row > span {
          font-weight: 500;
        }

        textarea {
          width: 100%;
          resize: vertical;
          overflow: hidden;
          border: 1px solid #3578e5;
          border-radius: 4px;
          background: #eee;
          color: #333;
          padding: 7px 8px;
          font: 18px/1.35 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
          white-space: pre-wrap;
          overflow-wrap: anywhere;
          word-break: break-word;
        }

        .flag-row {
          margin: 6px 0;
          font-size: 16px;
        }

        .flag {
          margin-left: 10px;
        }

        .flag.on {
          font-weight: 700;
        }

        .error {
          margin-top: 14px;
          padding: 10px 12px;
          border-radius: 8px;
          border: 1px solid #ffc2c2;
          background: #fff1f1;
          color: #a40000;
          font: 15px/1.45 system-ui, sans-serif;
        }

        .hint {
          margin-top: 10px;
          color: #666;
          font: 13px/1.4 system-ui, sans-serif;
        }

        @media (max-width: 720px) {
          .fmt-grid {
            grid-template-columns: 1fr;
          }

          .top {
            align-items: flex-start;
          }

          .top h1 {
            font-size: 22px;
          }

          .shortcut {
            display: none;
          }
        }
      </style>

      <main class="panel closed" role="dialog" aria-label="Decimal to Floating-Point Converter">
        <div class="top">
          <div>
            <h1>Decimal to Floating-Point Converter</h1>
            <div class="shortcut">快捷键：Alt + 4 隐藏 / 显示</div>
          </div>
          <button class="close" title="关闭，按 Alt + 4 可重新打开">×</button>
        </div>

        <div class="body">
          <section>
            <h2>Decimal</h2>
            <div class="input-line">
              <label for="fp-decimal">Enter a decimal number，例如：0.9、999999.9、3.1415、1.56e-11、4e20</label>
              <input id="fp-decimal" type="text" value="0.9" autocomplete="off" spellcheck="false" />
            </div>
            <button class="action convert">Convert</button>
            <button class="action clear">Clear</button>
          </section>

          <section class="opts">
            <div class="opts-title">Options:</div>

            <div class="opt-row">
              <span>Precision:</span>
              <label><input type="checkbox" data-opt="double" checked />Double</label>
              <label><input type="checkbox" data-opt="single" checked />Single</label>
            </div>

            <div>
              <div>Output formats:</div>
              <div class="fmt-grid">
                <label><input type="checkbox" data-opt="decimal" checked />Decimal</label>
                <label><input type="checkbox" data-opt="binary" checked />Binary</label>

                <label><input type="checkbox" data-opt="normDecimalScientific" />Normalized decimal scientific notation</label>
                <label><input type="checkbox" data-opt="normBinaryScientific" />Normalized binary scientific notation</label>
                <label><input type="checkbox" data-opt="normDecimalTimesPow2" />Normalized decimal times a power of two</label>
                <label><input type="checkbox" data-opt="intTimesPow2" />Decimal integer times a power of two</label>
                <label><input type="checkbox" data-opt="intTimesPow10" />Decimal integer times a power of ten</label>
                <label><input type="checkbox" data-opt="hexFloat" />Hexadecimal floating-point constant</label>

                <label><input type="checkbox" data-opt="rawBinary" checked />Raw binary</label>
                <label><input type="checkbox" data-opt="rawHex" />Raw hexadecimal</label>
                <label><input type="checkbox" data-opt="flags" checked />Flags</label>
              </div>
            </div>
          </section>

          <div class="hint">
            说明：脚本会把十进制字符串精确解析成分数，再按 IEEE-754 round-to-nearest, ties-to-even 舍入；
            float32 不会先经过 JS Number / double。
          </div>

          <div class="result"></div>
        </div>
      </main>
    `;

    const $ = (sel) => root.querySelector(sel);
    const $$ = (sel) => [...root.querySelectorAll(sel)];

    const panel = $('.panel');
    const input = $('#fp-decimal');
    const result = $('.result');

    function autoResizeTextareas() {
      root.querySelectorAll('.result textarea').forEach((textarea) => {
        textarea.style.height = 'auto';
        textarea.style.height = `${textarea.scrollHeight + 2}px`;
      });
    }

    function getOptions() {
      const opts = {};

      $$('input[type="checkbox"][data-opt]').forEach((cb) => {
        opts[cb.dataset.opt] = cb.checked;
      });

      return opts;
    }

    function convert() {
      try {
        const opts = getOptions();

        if (!opts.double && !opts.single) {
          throw new Error('请至少选择一种精度：Double 或 Single。');
        }

        const hasOutput =
          opts.decimal ||
          opts.binary ||
          opts.normDecimalScientific ||
          opts.normBinaryScientific ||
          opts.normDecimalTimesPow2 ||
          opts.intTimesPow2 ||
          opts.intTimesPow10 ||
          opts.hexFloat ||
          opts.rawBinary ||
          opts.rawHex ||
          opts.flags;

        if (!hasOutput) {
          throw new Error('请至少选择一种输出格式。');
        }

        const parsed = parseDecimalExact(input.value);
        const results = [];

        if (opts.double) {
          results.push(convertToIEEE(parsed, FP64));
        }

        if (opts.single) {
          results.push(convertToIEEE(parsed, FP32));
        }

        result.innerHTML = buildResultHtml(parsed, results, opts);
        autoResizeTextareas();
      } catch (err) {
        result.innerHTML = `<div class="error">${escapeHtml(err?.message || String(err))}</div>`;
      }
    }

    function openPanel() {
      panel.classList.remove('closed');

      setTimeout(() => {
        input.focus();
        if (!result.innerHTML.trim()) {
          convert();
        } else {
          autoResizeTextareas();
        }
      }, 0);
    }

    function closePanel() {
      panel.classList.add('closed');
    }

    $('.close').addEventListener('click', closePanel);

    $('.convert').addEventListener('click', convert);

    $('.clear').addEventListener('click', () => {
      input.value = '';
      result.innerHTML = '';
      input.focus();
    });

    input.addEventListener('keydown', (ev) => {
      if (ev.key === 'Enter') {
        convert();
      }
    });

    $$('input[type="checkbox"][data-opt]').forEach((cb) => {
      cb.addEventListener('change', convert);
    });

    document.addEventListener('keydown', (ev) => {
      const isAlt4 =
        ev.altKey &&
        !ev.ctrlKey &&
        !ev.shiftKey &&
        (
          ev.key === '4' ||
          ev.code === 'Digit4' ||
          ev.code === 'Numpad4'
        );

      if (isAlt4) {
        ev.preventDefault();

        if (panel.classList.contains('closed')) {
          openPanel();
        } else {
          closePanel();
        }
      }

      if (ev.key === 'Escape' && !panel.classList.contains('closed')) {
        closePanel();
      }
    });
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', createUI, { once: true });
  } else {
    createUI();
  }
})();
```


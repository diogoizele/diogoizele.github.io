---
title: "Dealing with Decimal Values in Controlled Inputs in React Native"
date: 2025-12-30 00:00:00 +0000
categories: [React, Masks, Controlled Inputs]
tags: [React, React Native]
description: "A brief experience on handling decimal values in controlled inputs using masks in React Native."
author: diogoizele
toc: true
image:
  path: /assets/img/002/cover.jpg
---

Recently, I faced a challenge when I needed to create a decimal numeric mask for a controlled input in a React Native app. As many developers know, handling decimal values in JavaScript is not an easy task, mainly because of floating-point precision issues.

In this case, the problem was not only about dealing with decimal numbers, but also about formatting the value while the user types. The input needed to keep the correct decimal separator, preserve intermediate states, and avoid breaking the cursor position.

The main requirement was to allow many decimals places. For example, a quantity with up to 6 digits, like `123.456789`. The value also needed to follow the Brazilian number format (i18n), using dots as thousands separators and comma as the decimal separator, like `1.234,56`.

To solve this, I decided to implement a custom mask by myself, instead of relying on external libraries. The idea was to keep full control over parsing, formatting and intermediate states.

---

## General Approach

The main idea is simple:
the input value is store as a **string**, not as a number.

We normalize the raw user input using a `parse` function, store this normalized string in the state, and then derive the formatted using a `format` function.

As first, this process may look, but in practice it starts with a simple React state:

```ts
const [inputValue, setInputValue] = useState('');
```
---

## Formatting the Value

The `format` function is responsible only for presentation. It does not change the meaning of the value, it only adds thousands separators and keeps the decimal part.


```ts
function format(text: string) {
  if (!text) return '';

  const [integerPart, decimalPart] = text.split(',');

  const formattedInteger = integerPart.replace(/\B(?=(\d{3})+(?!\d))/g, ".");

  if (text === ",") {
    return "0,";
  }

  if (text.endsWith(",")) {
    return `${formattedInteger},`;
  }

  if (decimalPart !== undefined) {
    return `${formattedInteger},${decimalPart}`;
  }

  return formattedInteger;
}
```
This function works as follows:
1. If the value is empty or falsy, return an empty string.
2. If the user types only a comma, return `0,` to avoid invalid stats like `,56`.
3. Split the value using the comma to separate integer and decimal parts.
4. Format the integer part using thousand separators.
5. If the value ends with a comma, keep it, allowing the user to continue typing decimals.
6. If a decimal part exists, preserve it and return integer and decimal parts separated by a comma.
7. Otherwise, return only formatted integer part.

> ❗ This function expects a string that uses a comma as the decimal separator.

---

## Parsing the Input

The `parse` function is responsible for normalization. It cleans the input, limits decimal places, and ensures the state always contains a valid representation.

```ts
function parse(text: string, decimalPlaces: number) {
  const cleaned = text.replace(/[^\d,]/g, '');

  const parts = cleaned.split(',');

  if (parts.length > 2) {
    return `${parts[0]},${parts.slice(1, decimalPlaces + 1).join('')}`;
  }

  const commaIndex = cleaned.indexOf(',');

  if (commaIndex === -1) {
    return cleaned;
  }

  return cleaned.slice(0, commaIndex + decimalPlaces + 1);
}
```

This function works as follows:
1. Remove any character that is not a digit or a comma.
2. Split the cleaned value by the comma.
3. If more than one comma is typed, keep only the first one and treat all remaining digits as part of the decimal portion.
4. If no comma exists, return the cleaned value.
5. If a comma exists, limit the number of decimal digits based on `decimalPlaces`.

> ⚠️ The `decimalPlaces` parameter controls how many decimal digits the user allowed to type.

--- 

## Using the Controlled Input

With both functions in place, the controlled input becomes straightforward:

```ts
const DECIMAL_PLACES = 6;

const App = () => {
  const [inputValue, setInputValue] = useState('');

  const handleChange = (text: string) => {
    const parsedValue = parse(text, DECIMAL_PLACES);
    setInputValue(parsedValue);
  };

  return (
    <TextInput
      value={format(inputValue)}
      onChangeText={handleChange}
      keyboardType="numeric"
      placeholder="Enter a number"
    />
  );
};
```
At this point, the input formats the value while typing, respects the decimal limit, and keeps a consistent internal state.

---

## Extra - Converting to a Domain Value

For domain logic, storing values as floating-point numbers is usually unsafe. To avoid precision issues, I recommend using `decimal.js`.

```ts
import Decimal from 'decimal.js';

function convertToDomainValue(text: string): Decimal | null {
  if (!text) return null;
  if (text.endsWith(',')) return null;

  const normalized = text
    .replace(/\./g, '')
    .replace(',', '.');

  return new Decimal(normalized);
}
```

This function:
1. Returns null if the value is empty or incomplete (ends with a comma).
2. Removes thousands separators (dots).
3. Replaces the decimal comma with a dot.
4. Creates a `Decimal` instance from the normalized string.

This allows safe calculations, storage, or API communication without floating-point errors.

--- 

## Full usage example

```ts
import React, { useState } from 'react';
import { TextInput } from 'react-native';
import Decimal from 'decimal.js';

import { format, parse, convertToDomainValue } from './maskUtils';

const DECIMAL_PLACES = 6;

const App = () => {
  const [inputValue, setInputValue] = useState('');
  const [decimalValue, setDecimalValue] = useState<Decimal | null>(null);

  const handleChange = (text: string) => {
    const parsedValue = parse(text, DECIMAL_PLACES);
    setInputValue(parsedValue);
  };

  const handleBlur = () => {
    const decimal = convertToDomainValue(inputValue);
    setDecimalValue(decimal);
  };

  return (
    <TextInput
      value={format(inputValue)}
      onChangeText={handleChange}
      onBlur={handleBlur}
      keyboardType="numeric"
      placeholder="Enter a number"
    />
  );
};
```

This approach is useful when you need strict control over decimal precision and user input behavior. For simpler cases, native formatters or libraries may be enough. However, when precision and intermediate states matter, handling the value as a string and converting it only at the domain level can be a reliable solution.

Thanks for reading! I hope this article was useful. You can find this and more input masking examples in this repository:
**Github**: [React Native Input Masks](https://github.com/diogoizele/react-native-input-masks)
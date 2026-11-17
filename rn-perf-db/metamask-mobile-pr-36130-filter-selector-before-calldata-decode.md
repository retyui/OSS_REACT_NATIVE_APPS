# perf(confirmations): avoid decoding transaction calldata in render path
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/36130>
- **Merged Date:** 2026-09-14
- **Category:** Rendering

## Problem
`parseStandardTokenTransactionData` tries the ERC20, ERC721, ERC1155 and USDC ABIs in sequence inside `try`/`catch`. Calldata matching none of them is the most expensive case (four failed parses plus four exception constructions), and that is exactly the calldata whose result is discarded. Three call sites decoded first and only afterwards checked whether it was an approval, and `useGasFeeToken` built an ERC20 `Interface` on every render. The PR reports no measurements.

## Solution
- Filter on the cheap 4-byte selector (a shared `APPROVAL_SELECTORS` set) before decoding in `useApproveTransactionData`, `parseApprovalTransactionData`, and `extractSpenderFromApprovalData`.
- In `useGasFeeToken`, replace the render-time `useMemo` that built the transfer transaction with a `getTransferTransaction` callback, invoked only on submit.
- The selector set is a strict superset of `APPROVALS_LIST`, so the pre-filter cannot reject calldata the downstream checks would accept.

## Code Pattern (Before vs. After)
```ts
// Before
const code = get4ByteCode(data);
const parsed = parseStandardTokenTransactionData(data);
return { fourByteCode: code, parsedData: parsed };

// After
const code = get4ByteCode(data);

if (!APPROVAL_SELECTORS.has(code)) {
  return { fourByteCode: code, isApproval: false, parsedData: undefined };
}

return {
  fourByteCode: code,
  isApproval: true,
  parsedData: parseStandardTokenTransactionData(data),
};
```

```ts
// Before (useGasFeeToken)
const transferTransaction = useMemo(
  () => /* builds ERC20 Interface and transfer tx on every render */,
  [/* ... */],
);

// After
const getTransferTransaction = useCallback(
  () => /* same work, only runs when called on submit */,
  [/* ... */],
);
```

## Takeaway
Run cheap guards (selector checks) before expensive decoding, and defer work whose result is only needed on a user action.

# Split the bill — study notes

Closed 2026-10-21. Read top to bottom.
The API is the HTTP contract. The HLD is the tables behind it.

## Product

People form a group. One person pays a bill outside the app. The others owe that person a share. Later someone pays that person back.
The app does not send money to the restaurant. An expense records a payment that already happened. A payback records money one member gave another member.

## How one call is written

Five lines, in this order.

1. Method
2. Path. The parent resource comes first. The path is nouns. The verb is the method.
3. Body
4. Response
5. Status

The phone does not see the service or the database. Those belong in the HLD.

## Calls

Create a group.

`POST /groups`
Body: `name`
Response: `id`, `name`
**201.** The caller becomes a member.

Add a person.

`POST /groups/{groupId}/members`
Body: `userId`
Response: `groupId`, `userId`
**201.** **403** when the caller is not in the group. **404** when the group does not exist.

Add an expense. One member already paid. The amount is split across members.

`POST /groups/{groupId}/expenses`
Body: who paid (`memberId`), `amount`, and the split (which members share that amount).
Response: expense `id`, `memberId`, `amount`
**201.** **404** when the group does not exist. **403** when that member is not in the group. A bad body, including a bad date, is **400**.

Read who owes whom.

`GET /groups/{groupId}/balances`
Body: none.
Response: one row per member, `userId` and `amount`. Positive means the group owes that person. Negative means that person owes the group.
**200.** An empty group is **200** with an empty list. **404** when the group does not exist.

Record a payback. This is not a new expense.

`POST /groups/{groupId}/payments`
Body: `fromUserId`, `toUserId`, `amount`, `clickId`
Response: payment `id`
**201** the first time. **404** when the group does not exist. **403** when either person is not in the group.

## Retry

The phone sent the payment. The response never arrived. The phone sends that same request again.
That second send is the same payment. A payment on another day is a new one.

The phone sends `clickId` on the same `POST`. The handler looks that id up in the repository. There is no second HTTP call. It is one service.

Same `clickId` with the same from, to, and amount: return the payment already stored. **200.**
Same `clickId` with a different amount or different people: **409.**
A new `clickId`: **201.**

Balance 0 does not mean this request was already saved. A partial payback leaves a balance. A retry must still not be stored twice. The click id is the test.

## Tables

`groups`: `id`, `name`

`expenses`: `id`, `group_id`, `payer_id`, `amount`

`balances`: `group_id`, `user_id`, `amount`
One row per person in that group. The group has no single total.

A pays 100 outside the app. Split is half with B. One transaction writes the expense and both balance rows.

| user_id | amount |
|---|---|
| A | +50 |
| B | -50 |

The 100 stays on the expense row.

A second expense, also paid by A, for 40, split in half, waits on A's balance row and on B's balance row. Both expenses are saved. The second reads the numbers the first commit left and adds its own half.

| user_id | amount |
|---|---|
| A | +70 |
| B | -70 |

The lock is that person's balance row in this group. It does not refuse the second expense.

B pays A 50. `POST /payments`. No new expense. A goes to 0. B goes to 0. The original expense row stays.

## One line

An expense records money already paid outside the app and updates each person's balance. A payback moves those same balance rows and does not create an expense. A retry is the same click id, checked in the repository inside the POST.

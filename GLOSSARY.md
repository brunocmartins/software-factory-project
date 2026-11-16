# Cadence

A course platform where students buy courses and instructors sell them.

## Language

**Student**:
A person who buys and takes courses.
_Avoid_: Learner, Customer, Buyer

### Refund Requests

**Refund Request**:
A Student's ask, made within the Refund Window, to get their money back on a Purchase. A Purchase has at most one Open Refund Request at a time, may have earlier Declined ones, and can have no further Refund Requests once one is Approved.
_Avoid_: Reversal, Chargeback, Cancellation, Dispute

**Open**:
A Refund Request that has been neither Approved nor Declined. The Student keeps access to the course while it is Open.
_Avoid_: Pending, Requested

**Approved**:
A Refund Request accepted for the full Refundable Amount. The Student loses access to the course, and a Refund is Issued.

**Declined**:
A Refund Request that has been turned down. Nothing is returned.

**Refundable Amount**:
The amount the Student actually paid for the Purchase. It is the same figure on every surface where the Student sees it.

**Refund Window**:
The 14 calendar days after a Purchase during which a Refund Request may be opened. A Refund Request opened inside the window may still be Approved or Declined after it closes.
_Avoid_: Cancel window

**Refund Eligibility**:
Whether a Purchase may have a Refund Request opened on it right now. It is judged only at the moment of opening.
_Avoid_: Cancellable, refundable (for the Purchase)

### Refunds

**Refund**:
The movement of the Refundable Amount back to the Student's card, which follows an Approved Refund Request. Its lifecycle is separate from the Refund Request's: the request records the decision, the Refund records the money.
_Avoid_: Reversal, Chargeback, Refunded Purchase

**Issued**:
A Refund the payment processor has been asked to make but has not yet confirmed.
_Avoid_: Pending, Initiated, Sent

**Completed**:
A Refund the payment processor has confirmed.
_Avoid_: Settled, Succeeded, Refunded

**Failed**:
A Refund the payment processor has rejected.

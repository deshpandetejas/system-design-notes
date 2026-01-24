

## Problem
As systems grow, performance issues often surface unexpectedly.
Scalability is not just about handling more traffic — it’s about handling growth **predictably**.

## Core Question
When demand increases, how does the system behave?

## Key Concepts

### Vertical Scaling
- Adding more power (CPU, RAM) to a single machine
- Simple to implement
- Hard limit exists
- Creates single points of failure

### Horizontal Scaling
- Adding more machines
- Requires coordination, load balancing, and statelessness
- More complex, but more resilient

## Trade-offs I Notice
- Vertical scaling is faster early but riskier long-term
- Horizontal scaling improves availability but increases operational complexity
- Many systems fail not because they can’t scale, but because scaling was not planned early

## Questions I Now Ask in Design Reviews
- Is this service stateful or stateless?
- What breaks first when load doubles?
- Is scaling this component linear or exponential in cost?
- What is the failure mode under peak traffic?

## Personal Takeaway
Scalability decisions are business decisions.
Over-engineering too early wastes money, but under-engineering creates outages and firefighting.
The real skill is knowing **when** to invest and **where**.


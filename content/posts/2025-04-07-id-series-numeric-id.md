+++
title = "ID Series: Dangerous Numeric ID [Part 1]"
date = "2025-04-07"
slug = "id-series-dangerous-numeric-id"
description = "The danger of numeric IDs"
tags = ["numeric id", "primary key", "database"]
categories = ["id series"]
+++

![What is the next number?](/images/posts/2025-05/2025-05-08-what-is-the-next-number.png)

When we launched Workstream in 2018 using Ruby on Rails, we followed the framework's convention: Use the integer as the primary key for database records. This choice simplified development and made our URLs more human-readable.

```sql
CREATE TABLE employees (
  id bigint NOT NULL PRIMARY KEY, /*👈 look at this */
  location_id bigint NOT NULL,
  company_id bigint NOT NULL,
  first_name varchar(255),
  last_name varchar(255)
);
```

Fast forward to our Series B funding, with middle and large enterprises joining our platform. These new customers brought heightened security requirements, including SOC2 compliance. Suddenly, our innocent default choice of numeric IDs faced scrutiny.

## Security Vulnerabilities

Workstream manages highly sensitive HR information, including I9 and W4 forms. Security is our top priority.

```sql
CREATE TABLE i9_forms (
  id bigint NOT NULL PRIMARY KEY,
  company_id bigint NOT NULL,
  data jsonb NOT NULL,
  created_at timestamp NOT NULL,
  updated_at timestamp NOT NULL,
);
```

As a multi-tenant SaaS application where each company has a unique ID, we relied on company scoping for data isolation:

```sql
SELECT * FROM employees WHERE company_id = 1000;
```

Besides default sql condition, we have implemented strict permission control for each request. However, we still need to consider the risk of human error. As engineering teams grow and the codebase expands, inexperience developers or junior developers may not correctly implement the API. With sequential numeric IDs, hackers don't need sophisticated techniques to access unauthorized data—they can simply increment numbers in API requests:

```text
GET companies/1000/i9_form/1322
GET companies/1000/i9_form/1323
GET companies/1000/i9_form/1324

GET companies/1000/w4_form/1000
GET companies/1000/w4_form/1001
GET companies/1000/w4_form/1002
```

## Numeric IDs Leak Business Metrics

Numeric IDs can inadvertently reveal business metrics to competitors. The competitors could create a resource in the morning and get the resource ID, then create another resource in the evening and get the resource ID.

```text
POST /resources  (9:00 AM)
=> id = 1001

POST /resources  (5:00 PM)
=> id = 2000
```

By analyzing the numerical range of exposed IDs, anyone can estimate that approximately 1,000 resources were processed in an 8-hour period. Scale this analysis across days or weeks, and competitors gain invaluable insights into our volumes, growth rates, and market penetration—all from observing numeric ID sequences that should have remained internal.

## Service Miscommunication

As organizations grow, Workstream evolved our Rails monolith to multiple microservices, and the inter-service calls increased. Using numeric IDs in inter-microservice communications could lead to serious data integrity issues.

Consider an employee record:

```text
first_name: Ryan
last_name: Lyu
location_id: 3010
company_id: 200
user_id: 1021
```

If a developer mistakenly uses the wrong ID type in an internal API call:

```
# Accidentally using location_id instead of user_id
GET /internal_api/users/3010
```

With numeric IDs, this request might silently succeed—returning data for an unintended user rather than failing fast. The error could cascade through the system, causing data corruption or inappropriate access without triggering obvious alerts.

## The Lesson

Numeric IDs provide convenience during the early stage of the startup. However, it becomes a weak point as the business grows.

In the following articles, we'll explore solutions to this problem, including UUID V4, UUID V7, and Type ID.

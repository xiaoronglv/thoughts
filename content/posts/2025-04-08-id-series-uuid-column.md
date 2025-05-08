+++
title = "ID Series: Extra UUID column for external Usage [Part 2]"
date = "2025-04-08"
slug = "id-series-extra-uuid-column-for-external-identifier"
description = "Adding an extra UUID column for external usage"
tags = ["UUID", "primary key", "database"]
categories = ["id series"]
+++

![](/images/posts/2025-05/2025-05-08-changing-wheels-while-still-driving.jpeg)

In the previous post [ID Series: Dangerous Numeric ID](/posts/id-series-dangerous-numeric-id), we discussed the dangers of using numeric IDs as primary keys. To mitigate the security risk, we decided to add an extra UUID column to all tables and use it as the external identifier.

## Goal

Our goal was to preserve the existing API endpoints and URL structures while transitioning to UUIDs as the public-facing identifiers.

Before:

```text
GET /candidates/1234567890
```

After:

```text
GET /candidates/123e4567-e89b-12d3-a456-426614174000
```

## Database Schema Modification

The first step involved altering the database schema. We added a uuid column for all tables and ensured it would be populated automatically for new records, along with an index for efficient lookups.

```ruby
class AddUuidToCandidates < ActiveRecord::Migration[6.0]
  def change
    # Add the new UUID column.
    add_column :candidates, :uuid, :uuid

    # Set the default value to a random UUID.
    change_column_default :candidates, :uuid, 'gen_random_uuid()'

    # Add an index for the new UUID column.
    add_index :candidates, :uuid
  end
end
```

## Router change

Beyond schema changes, the Rails application itself required updates to recognize and prioritize these new UUIDs.

**Router**

```ruby
Rails.application.routes.draw do
  resources :candidates, only: [:show], param: :uuid
end
```

**Controller**

The corresponding controller action needed to first attempt fetching a record by uuid. To maintain backward compatibility during the transition, especially if frontend systems had not yet been updated, it would fall back to searching by the numeric id.

```ruby
def show
  @candidate = Candidate.find_by(uuid: params[:uuid])

  # Fallback for legacy numeric IDs, if UUID lookup fails
  if @candidate.nil?
    @candidate = Candidate.find_by(id: params[:uuid])
  end
end
```

## Rollout strategy

1. Introduce the uuid column to all relevant tables.
2. Execute a backfill process to populate the uuid for all existing records.
3. Deploy the backend application changes (router and controller logic).
4. Update frontend applications to utilize UUIDs when fetching resources.

We did it!

## Emerging Drawbacks

However, this dual-identifier system soon presented new challenges. Each table now contained:

1. id, integer type, primary key
2. uuid, uuid type, unique identifier

This duality became a source of confusion within our microservice architecture. Developers, particularly those new to the team or the convention, might inconsistently reference the integer id or the uuid. For instance, one microservice might expect a user_id as an integer, while another interacting system might attempt to use the user's UUID.

This chaos requires each microservice to handle the fallback logic in their codebase, making the codebase more complex and harder to maintain:

```ruby
# Example of user session handling with dual ID types
if session[:user_id]
  user = if UUID.validate(session[:user_id])
            User.find_by(uuid: session[:user_id])
          else
            User.find_by(id: session[:user_id])
          end
  session[:user_id] = nil if user.blank? || user.disabled?
end
```

Our OpenAPI documentation also suffered. People have to specify the id is uuid, but numeric id is also supported and not recommended. This ambiguity complicated the API contract and developer experience:

```yaml
/internal_api/users/{id}:
    get:
        summary: Get User by id.
        description: Retrieves a user by their id.
        operationId: getUserById
        tags:
            - User
        security:
            - ApiKey: []
        parameters:
            - name: id
              in: path
              required: true
              description: |
                  The id of the user. \
                  The user's id can be either a numeric ID or a UUID. \
                  It is recommended to use the UUID value. 😞😞
```

## Next

While introducing the uuid column and implementing fallback logic does solove the security problem, it caused confusion, increased the code complexity, and reduced the development velocity. We need to find a better solution.

In the next post, we will discuss the better solution.

# Setup

Running locally:

Ruby 2.7.2

```
bundle install
```

```
rails db:drop db:create db:migrate db:seed
```

Running tests:

```
RAILS_ENV=test rails db:drop db:create db:migrate db:seed
```

```
rails test
```

# Exercise

The objective of these exercises is to start a conversation about the trade-offs when making code changes, specifically
around how code improvements impact code design and performance. We expect you to spend no more than 2-3 hours working on the
exercises. Only the **first challenge** is required to be completed via code, **be prepared to discuss each challenge** and how you would solve it.

## Challenge 1: Feature Request

The product owner has requested that the previous and next step numbers be included in the [Cases#index](http://localhost:3000/cases)
response for each case that is shown. Add `previous_step_number` and `next_step_number` attributes to the JSON output of
the [Cases#index](http://localhost:3000/cases) response.

Please consider that the current implementation for the JSON response already includes the attribute for `current_step_number`,
this is a good start. It is important to follow the patterns that already exist in the code, hint: see `current_step_number` method in [Case](blob/main/app/models/case.rb).

Example of the current implementation:
```json
{
  "results": [
    {
      "id": 1,
      "candidate_fullname": "Pancho Villa",
      "candidate_email": "panchovilla@revolution.mx",
      "due_date": "2021-07-22T21:28:35.628Z",
      "applicant_has_been_notified": false,
      "current_step_number": 0,
      "current_step_due_date": null,
      "current_step_all_requirements_complete": true,
      "current_panel_name": "Case is Being Created"
    },
    {
      "id": 2,
      "candidate_fullname": "Rosita Alvirez",
      "candidate_email": "rosita@revolution.mx",
      "due_date": "2021-10-22T21:28:35.654Z",
      "applicant_has_been_notified": false,
      "current_step_number": 0,
      "current_step_due_date": null,
      "current_step_all_requirements_complete": true,
      "current_panel_name": "Case is Being Created"
    },
    {
      "id": 3,
      "candidate_fullname": "Pancho Villa 1",
      "candidate_email": "panchovilla@revolution1.mx",
      "due_date": "2021-07-22T21:28:35.682Z",
      "applicant_has_been_notified": false,
      "current_step_number": 0,
      "current_step_due_date": null,
      "current_step_all_requirements_complete": false,
      "current_panel_name": "Case is Being Created"
    }
    ...
  ]
}
```
## Challenge 2: Performance Improvements

For performance improvements for the CasesController#index endpoint, one can optimize the queries and reduce unnecessary database calls. Here are some suggestions: 

1. Identify N+1 Query Issues:
With this, use 'Bullet' gem or Rails built in 'Bullet' feature to identify N+1 query problems. 

2. Use includes to Eager Load Associations:
In the CasesController#index action, use includes to eager load associations and avoid N+1 queries: 
def index
  @cases = Case.includes(:institution, :work_steps:[:panels]).all
  render 'cases/index', formats: [:json], handlers: [:jbuilder], status: 201
end

2. Select Only Necessary Fields: 
Select only the fields that are needed for rendering the JSON response, reducing the amount of data fetched from the database:
def index
  @cases = Case.includes(:institution, :work_steps).select(:id, :candidate_fullname, :candidate_email, :due_date, :applicant_has_been_notified, :current_step_number, :current_step_due_date, :current_step_all_requirements_complete,:current_panel_name,:previous_step_number,:next_step_number).all
  render 'cases/index', formats: [:json], handlers: [:jbuilder], status: 201
end

3. Limit the number of records loaded:
Use pagination or limit the number of records retrieved based on application's requirements: 
cases = Case.limit(20)
Loading a smaller number of records can improve performance, especially if you don't need to display a large number of records at once.
Adjust the value based on your application's requirements and performance considerations.

4. Caching
Use caching mechanisms to cache the JSON response or parts of it, especially if data doesn't change frequently. 
def index
  # ...
  # Check if the JSON response is already cached
  if (cached_result = Rails.cache.read('cases_index_json'))
    render json: cached_result
  else
    @cases = Case
      .includes(:institution, work_steps: [:panels])
      .select(
        :id,
        :candidate_fullname,
        :candidate_email,
        :due_date,
        :applicant_has_been_notified,
        :current_step_number,
        :current_step_due_date,
        :current_step_all_requirements_complete,
        :current_panel_name,
        :previous_step_number,
        :next_step_number
      )
      .all
    render 'cases/index', formats: [:json], handlers: [:jbuilder], status: 201
    # Cache the JSON response for future requests
    Rails.cache.write('cases_index_json', response.body, expires_in: 1.hour)
  end
end

5. Pagination
Consider implementing pagination to load a limited set of records per request
# cases_controller.rb
def index
  page = params[:page] || 1
  per_page = params[:per_page] || 10
  @cases = Case
    .includes(:institution, work_steps: [:panels])
    .select(
      :id,
      :candidate_fullname,
      :candidate_email,
      :due_date,
      :applicant_has_been_notified,
      :current_step_number,
      :current_step_due_date,
      :current_step_all_requirements_complete,
      :current_panel_name,
      :previous_step_number,
      :next_step_number
    )
    .paginate(page: page, per_page: per_page)  
  render 'cases/index', formats: [:json], handlers: [:jbuilder], status: 201
end


6. Database indexing
Ensure that the database tables are properly indexed, especially for columns used in queries.
class AddIndexesToCases < ActiveRecord::Migration[6.1]
  def change
    add_index :cases, :candidate_fullname
    add_index :cases, :due_date
    add_index :cases, :applicant_has_been_notified
    add_index :cases, :current_step_number
    add_index :cases, :applicant_was_notified_at
  end
end


7. Profiling
Use tools like Rails' built-in rack-mini-profiler or other profiling tools to identify bottlenecks and areas for improvement

## Challenge 3: Tech Debt (This is a nice to have but not required to be completed)

To address technical debt, improve the performance, maintainability and overall code quality. Below shows ways to improve the model: 
1.) Improve Error Handling
There is no proper error handling, not providing feedback when unexpected cases occur. Ways to add error handling includes logging errors or raising specific exceptions. 
def current_work_step
  @current_work_step ||= find_current_work_step || raise("No current work step found")
end
def previous_work_step
  return nil if current_work_step.step_number == 1
  @previous_work_step ||= find_previous_work_step || raise("No previous work step found")
end

2.) Improve Code Readability
Methods found are long and logic is complex. Refactor these methods. Turn to private methods some of the logic. 
def current_work_step
  @current_work_step ||= find_current_work_step
end

def previous_work_step
  return nil if current_work_step.step_number == 1
  @previous_work_step ||= find_previous_work_step
end

private

def find_current_work_step
  these_work_steps.find { |step| step.current }
end

def find_previous_work_step
  these_work_steps.find { |step| step.step_number == current_work_step.step_number - 1 }
end

3. Replace Conditional Rescues: 
Usage of 'rescue nil' in methods like  'current_work_step' and 'previous_work_step' make debugging hard. Use 'find by' or conditional statements instead to handle edge cases. 
def current_work_step
  @current_work_step ||= these_work_steps.find { |step| step.current }
end

def previous_work_step
  return nil if current_work_step.step_number == 1
  @previous_work_step ||= these_work_steps.find { |step| step.step_number == current_work_step.step_number - 1 }
end

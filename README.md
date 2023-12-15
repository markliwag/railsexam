# Exercise

## Challenge 1: Feature Request
Kindly look at models/case.rb and views/cases/index.json.jbuilder for the changes made to provide the feature request

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

4. Limit the number of records loaded:
Use pagination or limit the number of records retrieved based on application's requirements: 
cases = Case.limit(20)
Loading a smaller number of records can improve performance, especially if you don't need to display a large number of records at once.
Adjust the value based on your application's requirements and performance considerations.

5. Caching
Use caching mechanisms to cache the JSON response or parts of it, especially if data doesn't change frequently.

def index
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
    Rails.cache.write('cases_index_json', response.body, expires_in: 1.hour)
  end
end

7. Pagination
Consider implementing pagination to load a limited set of records per request

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


9. Database indexing
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


10. Profiling
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

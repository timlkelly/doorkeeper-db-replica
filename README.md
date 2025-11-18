# README

This repository recreates an issue with the Doorkeeper when Rails is configured
to use a read replica.

## Reproduction steps

1. `bin/setup`
1. Create OAuth application using https://oauthdebugger.com/
    1. Navigate to `localhost:3000/oauth/applications`
    1. Fill out the form with the following attributes
        ```
        name: test
        redirect_uri: https://oauthdebugger.com/debug
        confidential: true
        scopes: test-scope
        ```
1. Navigate to https://oauthdebugger.com/
    1. Use the following attributes for the form
        ```
        authorize_uri: http://localhost:3000/oauth/authorize
        redirect_uri: https://oauthdebugger.com/debug
        client_id: {oauth-app client id}
        scope: test-scope
        response_type: code
        response_mode: form_post
        ```
    1. Click send request and authorize the application
1. Create auth token. Add the relevant variables and run this `curl` command
locally
    ```
    curl -XPOST localhost:3000/oauth/token \
      -d "grant_type=authorization_code" \
      -d "code=${auth_code_from_oauth_debugger}" \
      -d "client_id=${client_id}" \
      -d "client_secret=${client_secret}" \
      -d "redirect_uri=https%3A%2F%2Foauthdebugger.com%2Fdebug" | jq
    ```
1. Repeat step 3 to create the error.

## Error details

Doorkeeper is attempting to create a database record in `GET` request to
`/oauth/authorize`. However, due to Rails' default [automatic database role
switching][automatic-role-switch], the read role was selected and this request
fails with an `ActiveRecord::ReadOnlyError - Write query attempted while in readonly mode: INSERT INTO "oauth_access_grants"`.

Here is a snippet of the relevant lines from the callstack.

```
/Users/tim/.rbenv/versions/3.4.7/lib/ruby/gems/3.4.0/gems/doorkeeper-5.8.2/lib/doorkeeper/oauth/authorization/code.rb:17:in 'Doorkeeper::OAuth::Authorization::Code#issue_token!'
/Users/tim/.rbenv/versions/3.4.7/lib/ruby/gems/3.4.0/gems/doorkeeper-5.8.2/lib/doorkeeper/oauth/code_request.rb:15:in 'Doorkeeper::OAuth::CodeRequest#authorize'
/Users/tim/.rbenv/versions/3.4.7/lib/ruby/gems/3.4.0/gems/doorkeeper-5.8.2/lib/doorkeeper/request/strategy.rb:8:in 'Doorkeeper::Request::Strategy#authorize'
/Users/tim/.rbenv/versions/3.4.7/lib/ruby/gems/3.4.0/gems/doorkeeper-5.8.2/app/controllers/doorkeeper/authorizations_controller.rb:139:in 'Doorkeeper::AuthorizationsController#authorize_response'
/Users/tim/.rbenv/versions/3.4.7/lib/ruby/gems/3.4.0/gems/doorkeeper-5.8.2/app/controllers/doorkeeper/authorizations_controller.rb:35:in 'Doorkeeper::AuthorizationsController#render_success'
/Users/tim/.rbenv/versions/3.4.7/lib/ruby/gems/3.4.0/gems/doorkeeper-5.8.2/app/controllers/doorkeeper/authorizations_controller.rb:9:in 'Doorkeeper::AuthorizationsController#new'
```

[automatic-role-switch]: https://guides.rubyonrails.org/active_record_multiple_databases.html#activating-automatic-role-switching

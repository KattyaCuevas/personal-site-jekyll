---
layout: post
title:  "Building and Consuming a JSON API with Rails and React"
date:   2020-11-19 19:52:28 -0500
categories: jekyll update
---
When I thought about how to build an API, I started to search for what is the "best way" to do it. I found that exists specifications to build an API, you can found it [here](https://jsonapi.org). There, you would found a list of "rules" which we have to follow about how to send and receive data in an API.

The next doubt I had, after knowing the "best way" to build an API, is how am I going to build that API with all those rules? It looks so much work to do. Well... That's not true! In Rails, it's easy with a gem called `jsonapi-resources`.

In this project, the frontend will be done with React. The last version of Rails (v.6.0.0), Rails comes with Webpacker integrated (gem to handle the integration Rails + Webpack). It will make easier for us to use React. 🙌

Consume the data from our API with React, it's not hard. But, formatting the data to send to the API could be complex. There is another library to do this! Also, this library is going to help you to validate the form data. This library is `Formik`.

## Let's start!

Versions of the tools we are going to use:

- Ruby 2.6.3
- Rails 6.0.0
- Yarn 1.17.3

### Setup Base Project

To create a new project with rails, we need to use the `rails new` command with the project name at the end.

We could also add some additional options. In this case, we will use `--database=postgresql` to use PostgreSQL as our database, `--skip-turbolinks` to avoid using `turbolinks` because we will handle routing in the frontend, and `--webpack=react` to make Rails generate the configuration for us to use React.js.

```bash
$ rails new my-app --database=postgresql --skip-turbolinks --webpack=react
```

Now, we're going to add a model called Post with 2 attributes: title and body. `title` is a string and `body` is a text. In Rails, the model represents the database tables. We can generate it with the `rails generate model` command followed by the model name with the attributes. The attributes should be separated by spaces and has the name and the type divided by `:`, like `title:string`. If we don't specify the type of the attribute, Rails will default to the type `string`.

The command generates a file with the model definition and a migration file that specifies the change to be made in the database, in this case, is the creation of the new table.

```bash
$ rails generate model Post title body:text
$ rails db:create
$ rails db:migrate
```

> _Note:_ We could also use `rails g` which is an alias of `rails generate`.

The `rails db:create` command creates the database of the project and the `rails db:migrate` command runs all the pending migrations since this is a new project it will run every migration.

We could add some seed data. To do it, we have to open the `db/seeds.rb` file and add the following lines:

```ruby
Post.create(title: "Post 1", body: "My first Post")
Post.create(title: "Post 2", body: "My second Post")
```

And to populate the database with our seed data, we need to run the command:

```bash
$ rails db:seed
```

---

In Rails projects, we should define the main route of the application this one is going to handle the path `/`. Go to `config/routes.rb` to define it and inside of the block `Rails.application.routes.draw`, add:

```ruby
root to: "home#index"

get "*path", to: "home#index", constraints: { format: "html" }
```

> _Note:_ The routes are defined as "home#index", this means the controller which is going to control the behavior is `HomeController` and the specified action in the controller is `index`.

We have to create the HomeController. First, let's create the `home_controller.rb` file in `app/controllers` folder. Inside, add the `index` action:

```ruby
class HomeController < ApplicationController
  def index; end
end
```

Every action renders a view, in this case using HTML. We need to create the view in `app/views/home` folder and name it `index.html.erb`. In this file, we have to render the script to load our React app.

```erb
<%= javascript_pack_tag 'posts' %>
```

The helper `javascript_pack_tag` will generate the following script tag:

```html
<script src="/packs/js/posts-a447c92837fa3b701129.js"></script>
```

> Note: The name of the pack is generated with a hash added at the end to let us cached it for a long time.

This script will load the pack `posts.jsx`. We have to create that pack in the `app/javascript/packs` folder:

```jsx
import React from "react";
import ReactDOM from "react-dom";
import App from "components/App";

document.addEventListener("DOMContentLoaded", () => {
  ReactDOM.render(
    <App />,
    document.body.appendChild(document.createElement("div"))
  );
});
```

We are going to use `@reach/router` to handle the routes in our React app. To install it, run:

```bash
$ yarn add @reach/router
```

Let's create the component `App.js` in `app/javascript/components` folder. We will use this component to manage the routes.

```jsx
import React from "react";
import { Router } from "@reach/router";
import PostList from "./PostList";

function App() {
  return (
    <Router>
      <PostList path="/" />
    </Router>
  );
}

export default App;
```

Here we will create our first route `/`, which is going to render the `PostList` component.

Now we are going to create the component `PostList.js` in `app/javascript/components` folder.

```jsx
import React from "react";

function PostList() {
  return <div>Hello from my React App inside my Rails App!</div>;
}

export default PostList;
```

Inside we are going to render a `div` to test our React App.

### Start the Server

We need to install `foreman` to run the React and Rails apps at the same time. We can install it with the command:

```bash
$ gem install foreman
```

We should create a `Procfile.dev` file in the root of the project. Inside it, add:

```
web: bundle exec rails s
webpacker: ./bin/webpack-dev-server
```

To start the server, we need to run the command:

```bash
$ foreman start -f Procfile.dev
```

### Create the API

To create our API following the JSON:API specification, we are going to use the gem `jsonapi-resources`. To use it, we have to add it to the `Gemfile` and install it running `bundle install`.

JSONAPI::Resources provides helper methods to generate correct routes. We'll add the routes for API in `config/routes.rb`, before `get "*path"`:

```ruby
namespace :api do
  jsonapi_resources :posts
end
```

> _Note:_ `namespace :api` is going to generate routes with `/api` before the route. Eg. `/api/posts`.

We're going to create the `ApiController`, to extend the controller from the `ActionController::API` module of Rails, and also we're going to include the `JSONAPI::ActsAsResourceController` from JSONAPI::Resources.

```ruby
class ApiController < ActionController::API
  include JSONAPI::ActsAsResourceController
end
```

Now we need to create the `PostsController`. We should create it inside a folder named `api` because our routes config is going to search for an `Api::PostsController` class.

```ruby
class Api::PostsController < ApiController
end
```

`jsonapi_resources :posts` require a `PostResource` class defined. We have to create `PostResource` in `app/resources/api/post_resource.rb`.

```ruby
class Api::PostResource < JSONAPI::Resource
  attributes :title, :body
end
```

Here, we define the attributes and relationships we want to show as part of the resource.

To see how our response looks like, go to `localhost:5000/api/posts`.

### Consume the API

We will make the React app consume our API. First, let's only read the data. Edit the `PostList` component to fetch the list of posts.

```jsx
import React, { useEffect, useState } from "react";

function PostList() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const requestPosts = async () => {
      const response = await fetch("/api/posts");
      const { data } = await response.json();
      setPosts(data);
    };
    requestPosts();
  }, []);

  return posts.map(post => <div>{post.attributes.title}</div>);
}

export default PostList;
```

Inside a `useEffect`, we will do the fetch to `/api/posts` and save the response in the state of the component.

Now, let's create the form to add more posts. But first, we have to add `formik` as a dependency in the React app.

```bash
$ yarn add formik
```

We are going to create a new component to show the form, let's call it `AddPost.js`. In this component, we are going to make a POST method to `/api/posts` with the correct format of data to create a new post.

```jsx
import React from "react";
import { navigate } from "@reach/router";
import { Formik, Field, Form } from "formik";

function AddPost() {
  const handleSubmit = values => {
    const requestPosts = async () => {
      // We get the CSRF token generated by Rails to send it
      // as a header in the request to create a new post.
      // This is needed because with this token, Rails is going to
      // recognize the request as a valid request
      const csrfToken = document.querySelector("meta[name=csrf-token]").content;
      const response = await fetch("/api/posts", {
        method: "POST",
        credentials: "include",
        headers: {
          "Content-Type": "application/vnd.api+json",
          "X-CSRF-Token": csrfToken
        },
        body: JSON.stringify({ data: values })
      });
      if (response.status === 201) {
        navigate("/");
      }
    };
    requestPosts();
  };

  return (
    <div>
      <h2>Add your post</h2>
      <Formik
        initialValues={\{
          type: "posts",
          attributes: {
            title: "",
            body: ""
          }
        }}
        onSubmit={handleSubmit}
        render={() => (
          <Form>
            <Field type="text" name="attributes.title" />
            <Field type="text" name="attributes.body" />

            <button type="submit">Create</button>
          </Form>
        )}
      />
    </div>
  );
}

export default AddPost;
```

Finally, we need to add the route `/add` in our React app.

```jsx
import React from "react";
import { Router } from "@reach/router";
import PostList from "./PostList";
import AddPost from "./AddPost";

function App() {
  return (
    <Router>
      <PostList path="/" />
      <AddPost path="/add" />
    </Router>
  );
}

export default App;
```

If we go to `localhost:5000/add`, we will see the form. If we fill the fields and click on Submit, it will create a new post and will navigate automatically to `localhost:5000/`, where we will see our new post as part of the list.

If we reload the page, the React app will fetch our post again with the new post we just created.

That's how we can create an application with Rails + React, following the JSON:API spec.

I would love any feedback about the post or the libraries used here. ❤️

#Trello clone with Phoenix and React (第十二章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#Deploying on Heroku

We finally made it. After 11 parts we've learned how to setup a new Phoenix project with Webpack, React and Redux. We have created a secure authentication system based on JWT tokens, created migrations for the necessary schemas for our database, coded socket and channels for realtime features and built a GenServer process to keep track of connected board members. Now is time to share it with the world by deploying it on Heroku. Let's do this!

##Setting up Heroku

Before going any further we'll assume we already have a Heroku account and the Heroku Toolbet installed. For deploying Phoenix applications on Heroku we need to use two different buildpacks, so lets create the new application using the multi-buildpack:
```
$ heroku create phoenix-trello --buildpack https://github.com/ddollar/heroku-buildpack-multi
```
This will create our new application on Heroku and add the git remote heroku repository that we'll use for deploying. As just said before, we need two different buildpacks for a Phoenix application:

*  heroku-buildpack-elixir: Main buildpack for Elixir applications.
*  heroku-buildpack-phoenix-static: For static assets compilation.
To add both of them lets create a .buildpacks file and add both of them:
```
# .buildpacks

https://github.com/HashNuke/heroku-buildpack-elixir
https://github.com/gjaldon/phoenix-static-buildpack
```
If we need to change any aspect regarding the new Elixir production environment, we can do it by adding a elixir_buildpack.config file:
```
# elixir_buildpack.config

# Elixir version
elixir_version=1.2.3

# Always rebuild from scratch on every deploy?
always_rebuild=true
```
In our case we are specifying the Elixir version and also forcing the environment to rebuild everything, included dependencies, on every deployment. The same can be done for static assets by adding a phoenix_static_buildpack.config file:
```
# phoenix_static_buildpack.config

# We can set the version of Node to use for the app here
node_version=5.3.0

# We can set the version of NPM to use for the app here
npm_version=3.5.2
```

In this case we are specifying the node and npm versions we need for Webpack to build our static assets. Finally we have to create a compile file where we'll set how to compile our assets after every new deployment:
```
# compile

info "Building Phoenix static assets"
webpack
mix phoenix.digest
```
Note that we run the phoenix.digest mix task after the webpack build to generate the digested and compressed versions of the assets.

#Setting up our production environment

Before deploying for the first time, we need to update the prod.exs file with some necessary configuration changes:
```elixir
# config/prod.exs

use Mix.Config
# ...

config :phoenix_trello, PhoenixTrello.Endpoint,
  # ..
  url: [scheme: "https", host: "phoenix-trello.herokuapp.com", port: 443],
  # ..
  secret_key_base: System.get_env("SECRET_KEY_BASE")

# ..

# Configure your database
config :phoenix_trello, PhoenixTrello.Repo,
  # ..
  url: System.get_env("DATABASE_URL"),
  pool_size: 20

# Configure guardian
config :guardian, Guardian,
  secret_key: System.get_env("GUARDIAN_SECRET_KEY")
```
Basically what we are doing is enforcing it to use our Heroku application's url and enforce the SSL connection provided. We are also using some environment variables to configure the secret_key_base, database url and guardian's secret_key. The database url will be automatically created by Heroku once we deploy it for the first time, but for the other two we need to generate them and add them using the command line:
```
$ mix phoenix.gen.secret
xxxxxxxxxx
$ heroku config:set SECRET_KEY_BASE="xxxxxxxxxx"
...
...

$ mix phoenix.gen.secret
yyyyyyyyyyy
$ heroku config:set GUARDIAN_SECRET_KEY="yyyyyyyyyyy"
...
...
```
And we are ready to deploy!

##Deploying

After committing all this changes to our repository, we can deploy the application by simply running:
```
$ git push heroku master
...
...
...
```
If we take a look to the console output we can see how both buildpacks do their job by installing Erlang and Elixir with their necessary dependencies as well as node and npm among other tasks. Finally we need to run the migration in order to create the database tables:
```
$ heroku run mix ecto.migrate
```
And that's it, our application is deployed and ready to go!

##Conclusion

Deploying a Phoenix application on Heroku is pretty easy and straightforward. It might not be the best solution around, but for a demo application like this it works really well. I hope you have enjoyed building and deploying this application as much as I have. While writing the whole series I've made a lot of changes to the final codebase, correcting some stuff and adding a lot more features. If you want to check them don't forget to visit de demo or fork the repository:

Live demo Source code
Thanks for reading and for the support :)

Happy coding!
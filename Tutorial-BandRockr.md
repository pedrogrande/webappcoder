# BandRockr

This tutorial steps you through the creation of a web app that is perfect for a rock band. We will make it a generic site so that any band can use it.


We will use:
+ Omniauth to allow your users to log in with Facebook
+ Heroku to upload your app to the internet.
+ refactoring to tidy up our code



### Step 1.

Make sure your command line is using the correct RVM.

`rails -v` will tell you if you are using the right version. If it says that Rails is not installed, then enter the RVM command:

```
rvm use ruby-2.0.0-p247@rails4
```

If you get an error that says you need to use `--create` do this:

```
rvm use ruby-2.0.0-p247@rails4 --create
gem install rails --version=4.0.0 --no-docs --no-ri
```


### Step 2. 

Go to the [Rails Composer](http://railsapps.github.io/rails-composer/) app to copy the url you will need to create the app.

Open your command line/terminal, go to your apps folder and create the new app

```
rails new bandrockr -m https://raw.github.com/RailsApps/rails-composer/master/composer.rb
```

Respond to the questions as follows:

1. (3) Build your own application
2. (2) Thin
3. (2) Thin
4. (1) SQLite
5. (1) ERB
6. (1) Test::Unit
7. (1) None
8. (1) None
9. (1) None
10. (2) Twitter Bootstrap
11. (2) Twitter Bootstrap (SASS)
12. (4) Sendgrid
13. (3) Omniauth
14. (1) Facebook
15. (2) CanCan with Rolify
16. (2) SimpleForm
17. (4) Home Page, User Accounts, Admin Dashboard
18. y (Ban spiders)
19. y (Create a Github repo)
20. n (Don't use application.yml)
21. y (Reduce logs)
22. y (Use better_errors)
23. n (Don't create project specific gemset)

### Step 3.

Now go into the folder that we just created in your command line

```
cd bandrockr
```

Open the directory in Sublime `subl .`

Because the RailsComposer has not yet been properly updated for Rails 4, we need to remove the *attr_accessible* lines from the User model file:

Delete lines 3 and 4 from *app/models/user.rb*
```
  attr_accessible :role_ids, :as => :admin
  attr_accessible :provider, :uid, :name, :email
```


Test that the app works by running your server `rails s` and opening your browser to [localhost:3000](localhost:3000)

If you get an error, you might need to run `bundle`.

### Step 4.

#### Setting up Facebook login

Go to the [Facebook developers](https://developers.facebook.com/) site and click on **Apps** in the top navbar.

Find the *Create new app* button and click it.

Enter a unique app name - I used *AutorockrPete* - it won't accept a name that is already in use somewhere else.

I chose *Entertainment* as the category and click **Continue**.

Your new Facebook app is in *Sandbox* mode - that means that only you will be able to use it. When we put your site live to production, we will take it out of Sandbox mode - but for now leave it.

In the section *Select how your app will integrate with Facebook*, click on *Website with Facebook login*

Then enter `http://localhost:3000` in the field that is shown.

Scroll down and click *Save changes*

Leave the browser for a minute and go to Sublime where your *bandrockr* code is and open *config/initializers/omniauth.rb* 

You will see this:
```
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :facebook, ENV['OMNIAUTH_PROVIDER_KEY'], ENV['OMNIAUTH_PROVIDER_SECRET']
end
```

You can see that there are two environmental variables (enclosed in ENV['']) so we need to put these values into our command line window. You get these values from the Facebook app you just created and you will find them at the top of the page. The provider key is the App ID and the provider secret is the App Secret.

Go to the terminal window where your bandrockr app is running and stop the server (Control + C).
Enter the following (replacing the relevant parts with your own app id and secret):
```
export OMNIAUTH_PROVIDER_KEY=XXXXXXXXXXXXXXX OMNIAUTH_PROVIDER_SECRET=XXXXXXXXXXXXXXX
```

Then restart your server (`rails s`)

Reload `localhost:3000` in your browser and click on the Login button to see if it all works.

You should redirected to a Facebook page to give your authorisation, and then redirected back to your site and be logged in. You should also now see your name on the Home page.

If you click on your name, you will see the user's show page with your name and email address. You will also see the *Admin* button in the navbar - as the first user to sign in, you have been given the role of Admin.

Facebook gives us more information about the signing in user which I will show you how to access later in the tutorial.


### Step 5.

Now that we have the basic site set up, we will start to add functionality.

But let's first think about the application design.

I was thinking the data models that make sense are:

+ BandProfile - where the band can put in their name, info
+ Member - info about each band member
+ Link - a collection of the band's and band members' social media links
+ Album - links, image, info
+ Track - links, image, info
+ Photo - image, info
+ Gig - info about their upcoming/past gigs

There is a lot more we could do but lets start with this.

As we discussed last weekend, this is a simple app in regard to data models. The only real data relationships required here are:

+ A BandProfile has many Links, a Link belongs to BandProfile (one to many)
+ A Member has many Links, a Link belongs to Member (one to many)
+ An Album has many tracks, a Track belongs to an album (one to many)

You will notice that a link could belong to either the band or a band member. We could create a BandLink model and a MemberLink model (instead of a single Link model) but lets try it this way and see how it works.

To be fair, a track could belong to more than one album but let's leave it like this for now.

Let's create our scaffolds.

```
rails g scaffold BandProfile name info:text
```

```
rails g scaffold Member name info:text image
```

```
rails g scaffold Link title url band_profile:references member:references
```

```
rails g scaffold Album title release_date:date info:text buy_link
```

```
rails g scaffold Track title album:references info:text buy_link
```

```
rails g scaffold Photo image caption
```

```
rails g scaffold Gig title date:date start_time:time finish_time:time location street_address suburb tickets_url
```

check your files in *db/migrate* to make sure they look right :)

now run `rake db:migrate`

Now add the relationships into the model files:

*app/models/album.rb*
```
class Album < ActiveRecord::Base
	has_many :tracks
end
```

*app/models/member.rb*
```
class Member < ActiveRecord::Base
	has_many :links
end
```

*app/models/band_profile.rb*
```
class BandProfile < ActiveRecord::Base
	has_many :links
end
```

### Step 6.

Now we will set up the Admin page so that the band can enter their information.

We will put the info about the band on the Admin page (and show an edit button) but if they have not entered their information yet, we will have a button to Create their band profile. 

The current *admin* page (created by RailsComposer) is actually the User index page. We will now create our own admin page. In your terminal:

```
rails g controller admin index
```

Go to *config/routes.rb* and change the new line at the top from:
```
get "admin/index"
```

to 

```
get "admin" => "admin#index"
```

Now let's change the link in the navbar so that the Admin button actually takes you to the admin page.

In *app/views/layouts/_navigation.html.erb*, find the following line (probably line 15):
```
<%= link_to 'Admin', users_path %>
```

And change it to:
```
<%= link_to 'Admin', admin_path %>
```

Go to your browser, refresh the page and check that the Admin button takes you to the new admin page.

To show the band's info on the admin page, we need to add instance variables to the admin controller and its index method (action).

*app/controllers/admin_controller.rb*

```
class AdminController < ApplicationController
  def index
  	@band_profile = BandProfile.first
  	@links = Link.all
  	@members = Member.all
  	@gigs = Gig.all
  	@albums = Album.all
  end
end
```
As the band should only have one profile - we will need to bring in only the 'first' profile created, and also prevent more from being created.

Let's edit the Admin page. *app/views/admin/index.html.erb*

Here, we are:
+ adding an if/else statement
+ if a band profile does not exist, show the button to Add a profile
+ if a band profile **does** exist, show the info the band has provided
+ if there are links associated with the band profile, show them
+ show a button to add links
+ if band members have been added, show them and their info
+ show a button to add band members
+ if gigs have been added, show the info about them
+ show a button to add a gig
+ if photos have been added, show a button to the photo gallery/list
+ show a button to add a photo
+ if an album has been added, show the title and tracks on it
+ show an edit button next to each listed track
+ show a button to add a track
+ show a button to add an album

[Click here to see the admin/index.html.erb file](https://github.com/pedrogrande/bandrockr/blob/master/app/views/admin/index.html.erb)

### Step 7.

Let's show some stuff on the home page.

+ The most recent photo uploaded
+ The next gig
+ The latest album
+ An embedded track (the most recent)
+ Band info
+ Links
+ Twitter feed

Starting with the *app/controllers/home_controller.rb* file, we will bring in the data we will need to display in the view.

```
class HomeController < ApplicationController
  def index
    @photo = Photo.first
    @gig = Gig.next
    @album = Album.first
    @track = Track.first
    @band_profile = BandProfile.first
    @links = @band_profile.links
  end
end
```

Then apply some default scopes to our models so that the first record returned is the last one added. Except for Gig, where we will create a method in our Gig model so that it returns the next gig to the controller.

*app/models/photo.rb*
```
class Photo < ActiveRecord::Base
	default_scope { order("created_at DESC") }
end
```
This default scope means that every time a database call for photos is made, they are ordered by the created_at field in descending order.

*app/models/gig.rb*
```
class Gig < ActiveRecord::Base
	def self.next
		where("date > ?", Date.today).order("date ASC").first
	end
end
```

*app/models/album.rb*
```
class Album < ActiveRecord::Base
	has_many :tracks

	def self.latest
		order("release_date DESC").first
	end
end
```

*app/models/track.rb*
Work it out yourself :) Look at photo.rb for clues.


We'll be editing *app/views/home/index.html.erb*












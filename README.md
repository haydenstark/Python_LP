# Python_LP
 Code summary of Python/Django Live Project
 
 ## [Dine-List demo video](https://youtu.be/mXxXeUksQck)
 
 * [Search Functionality](#search-functionality)
 * [Zomato API](#zomato-api)
 * [Saving API Data: MyList](#saving-api-data-mylist)
 * [Front End](#front-end)
 * [Skills Learned](#skills-learned)

## Intro
During a two week sprint, I worked solo creating an app using Django to display a searchable collection of restaurants - called 'Dine-List'. Within this app, the user has the ability to search nearby or via city, save any found restaurant locally (to 'MyList'), edit some fields of the restaurant and add their own rating and comments, and add their own restaurant alongside others locally (in the event they couldn't find a known restaurant). This was my first experience using APIs and this app uses three in total - Zomato API for the restaurants, HERE Maps API to display restaurants in a map view, and IP-api for current user geolocation - as well as Beautiful Soup to datascrape the web and populate any restaurant images not received from Zomato's API. While this project seemed daunting at first, I quickly became entrenched in it and found I still have a ton of ideas I'd like to eventually go back and add in. In all, I was able to finish nine user stories and below are further descriptions of what I was able to accomplish within those two weeks.

## Search Functionality
The first issue I ran into was realizing there are multiple cities with the same name. Thus the city search on the home page is broken into two parts - first query to Zomato's API to display potential matching cities, and then a second query to pull restaurants from the specified city.


        #search by city on home page, this is a two-step process, we first need to query zomato for city id (multiple cities exist with the same name)
        def Restaurant_api(request):
            if request.method == 'POST':
                city = request.POST.get('userCity')     #grab user input
                response = requests.get('https://developers.zomato.com/api/v2.1/cities?q={}'.format(city), headers=auth)
                city_data = response.json()["location_suggestions"]
                if city_data == []:                     #if the response came back empty
                    messages.error(request, "Your city didn't match any in our database - try one more time!")
                    return redirect('RestaurantHome')   #sends ^this alert message and keeps user on home page
                else:                                   #otherwise if the response has data in it
                    cities = {}                         #new dictionary to store incoming data
                    i = 0
                    while i < len(city_data):           #cycle through all of the returned data and only display
                        id = city_data[i]["id"]         #   city names and their corresponding id which we will need
                        name = city_data[i]["name"]     #   to query the api again
                        cities.update({id: name})       #actually appends, rather than update, so existing entries are not overwritten
                        i += 1

                    content = {
                        'go': True,                     #this triggers the jQuery to open the "did you mean" modal
                        'cities': cities,               #carries our new dictionary into modal
                    }
                    return render(request, 'RestaurantApp/RestaurantApp_home.html', content)


        #search result from user input, only runs after user selects specific city
        def Restaurant_search(request, key):
            id = key                                #key is specified by which city the user selects
            locresponse = requests.get('https://developers.zomato.com/api/v2.1/location_details?entity_id={}&entity_type=city'.format(id), headers=auth)
            lat = locresponse.json()["location"]["latitude"]            #accessing zomato api for city's latitude and longitude
            lon = locresponse.json()["location"]["longitude"]           #   needed for filtering
            response = requests.get('https://developers.zomato.com/api/v2.1/search?entity_id={}&entity_type=city'.format(id), headers=auth)
            content = iterateResponse(response, lat, lon)                #function located lower down
            return render(request, 'RestaurantApp/RestaurantApp_search.html', content)

While this search functionality works for the interim, in the future I would instead populate the back-end of the home page with all potential searchable cities and create a suggestive search bar as the user is typing.
My preferred method of searching was the 'near me' functionality. The biggest hurdle I had to overcome with finding the user's location was that the app was being run completely locally (considering this project wasn't going to go live). I fortunately found IP-api, a free geolocation API, that automatically pulled your IP address without needing to provide it and, though it works, it's a rather rough guess - it's not completely accurate, but does find the general area you are currently in. 

## Zomato api
The bulk of the app - in all, I collected eighteen different variables from the API for all needed information.

        def iterateResponse(response, lat, lon):
            #each function's response variable queries the api a little bit differently, this is where they all catch up the same;
            city_data = response.json()["restaurants"]
            restaurants = {}                                                        #setting up dictionary to store all our needed data
            i = 0
            while i < len(city_data):                                               #cycle through each restaurant we received from api
                id = city_data[i]["restaurant"]["id"]                               #this is all the data we are requesting from the api each time
                name = city_data[i]["restaurant"]["name"]
                url = city_data[i]["restaurant"]["url"]
                img = city_data[i]["restaurant"]["thumb"]
                address = city_data[i]["restaurant"]["location"]["address"]
                latitude = city_data[i]["restaurant"]["location"]["latitude"]
                longitude = city_data[i]["restaurant"]["location"]["longitude"]
                cuisines = city_data[i]["restaurant"]["cuisines"]
                hours = city_data[i]["restaurant"]["timings"]
                avgfortwo = city_data[i]["restaurant"]["average_cost_for_two"]
                pricerange = city_data[i]["restaurant"]["price_range"]
                highlights = city_data[i]["restaurant"]["highlights"]
                rating = city_data[i]["restaurant"]["user_rating"]["aggregate_rating"]
                rating_text = city_data[i]["restaurant"]["user_rating"]["rating_text"]
                votes = city_data[i]["restaurant"]["user_rating"]["votes"]
                menu = city_data[i]["restaurant"]["menu_url"]
                phone = city_data[i]["restaurant"]["phone_numbers"]
                establishment = city_data[i]["restaurant"]["establishment"]
                if img == '':              #if the api returned nothing for img
                    try:
                        useragent = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102                                              Safari/537.36'}
                        shortURL = url.split('?')[0]                                                       #(^user agent is needed to gain access)
                        page = requests.get(shortURL + '{}'.format('?amp=1'), headers=useragent, allow_redirects=False)
                        soup = BeautifulSoup(page.content, 'html.parser')           #use beautiful soup to data scrape the site with the
                        try:                                                        #    url we already have and add an image
                            src = soup.findAll('amp-img')[1]['src']
                            img = src.split('?')[0]                                 #this split is just to clean the jpg of any filters/cropping
                        except:
                            src = soup.findAll('amp-img')[2]['src']
                            img = src.split('?')[0]
                    except :
                        pass
                restaurants.update({i: {
                    'id': id, 'name': name, 'url': url, 'img': img, 'address': address, 'latitude': latitude,
                    'longitude': longitude, 'cuisines': cuisines, 'hours': hours, 'avgfortwo': avgfortwo,
                    'pricerange': pricerange, 'highlights': highlights, 'rating': rating, 'rating_text': rating_text,
                    'votes': votes, 'menu': menu, 'phone': phone, 'establishment': establishment,
                }})
                i += 1
            cityresponse = requests.get('https://developers.zomato.com/api/v2.1/cities?lat={}&lon={}'.format(lat, lon), headers=auth)
            city = cityresponse.json()['location_suggestions'][0]['name']              #this is to be able to display the city name at the top of the page
            content = {'restaurants': restaurants, 'city': city, 'lat': lat, 'lon': lon,}
            return content
            
To avoid information overload, a lot of the data is tucked into a details panel on each restaurant's card that is only displayed when an 'info' button is clicked. Latitude and longitude aren't necessarily user-friendly and are not displayed, but needed to populate the map view. One functionality I would like to eventually add is creating an href on each map item to relocate the user to that specified restaurant in the List View. Filters are also available at the top of the search results page for the user to narrow down what they're looking for (restaurant name, dish, cuisine type, order by rating or price, and show restaurants within a specified radius) - this is sent as another query to Zomato's API and re-renders the search results page.

## Saving api data: MyList
This is the section that has the most potential for so many additional features! It could very easily be transformed into a social networking system emulating a facebook or twitter feed. Future ideas are adding a login system, allowing users to share restaurants with each other, a poll that asks you and your friends what cuisine sounds best for dinner tonight and showing a list of suggested restaurants based on the group's top choice... Showing restaurants that are currently being frequented by your friends or popular in your area, the list goes on and on! For now, the site currently allows the user to add their own rating and comments, known as 'MyRating' and 'MyComments', without mixing in with the rating received from the api. One hurdle was allowing the user to add their own restaurant locally. In order to not overwhelm the user with form fields, I limited the amount of input fields, but in effect had to incorporate a lot of 'if none' logic for the restaurant cards displayed in 'MyList'. Essentially, since a user-added restaurant was going to be missing a significant amount of information compared to ones received from the Zomato API, there are a lot of 'if' statements for the system to run through to avoid displaying 'None' across the entire restaurant card, but still allows information from the API to be displayed.
Below is a very small portion of a restaurant card (the menu button and MyRating - user rating separate from the api rating)

    <!-- MYRATING AND MENU BUTTON --><br>
     {% if restaurant.user_rating %}
      <div class="row">
           <div class="col-md-6 text-center">
               <div class="text-muted font-weight-bold" style="font-size: 1.25em;">MyRating</div>
               <div class="text-muted"><b><span id="myrating{{ restaurant.id }}" title="{{ restaurant.user_rating }}" style="font-size: 1.6em;"></span></b></div>
               <div class="text-muted" style="font-size: .9em;">You gave this restaurant: &nbsp;<span class="text-nowrap"><span class="font-weight-bold" style="font-size: 1.15em;">{{ restaurant.user_rating }}</span> stars!</span></div>
           </div>
           {% if restaurant.menu %}
           <div class="col-md-6 text-right">
               <div class="btn-group w-50 mt-4" role="group">
                   <button type="button" class="btn btn-lg btn-outline-danger font-weight-bold" onclick="window.location.href='{{ restaurant.menu }}'">Menu</button>
               </div>
           </div>
           {% endif %}
      </div>
      {% elif restaurant.menu %}
      <div class="col-md-6 offset-md-6 text-right pr-0">
           <div class="btn-group w-50 mt-4" role="group">
                <button type="button" class="btn btn-lg btn-outline-danger font-weight-bold" onclick="window.location.href='{{ restaurant.menu }}'">Menu</button>
           </div>
       </div>
       {% endif %}

## Front End
For this project, I utilized Bootstrap for my front-end. I've used bootstrap before, but this project gave me a lot more insight and a better understanding of how their layout system works and how to better customize their classes to fit my ideas for the presentation of the site. I strongly encourage to check out the demo video - seeing presentation is better than trying to read presentation!

## Skills Learned
* Better understanding of Django
* Better understanding of Bootstrap
* Utilizing Beautiful Soup
* Communicating with and pulling data from apis
* User-error proofing form fields client-side
* Error proofing form submissions back-end
* Cleaning data before committing it to database
* Communicating with a team if an idea is the appropriate approach before taking action
* Better understanding and utilizing responsive design
* Error proofing dynamically populated web pages
* Understanding system error messages more rapidly
* Passing multitude of variables throughout site and reusing those variables when needed
* Improved time management

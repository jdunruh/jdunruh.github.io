---
layout: post
title: Exploring Om.next Part 1
---

At the 2014 clojure/conj, the very first talk was about data driven applications, both back end and front end. I decided to explore clojurescript, Om and
the data driven application idea by building a simple front end application in [Om](https://github.com/omcljs/om). It was really interesting, and I realized the claims about the flexibility
and ease of development claims for data driven appplications were true. Om was interesting, and I found I liked the React model. On the other hand, there
were some issues with Om. The syntax was somewhat difficult and obscure, as was the handling of cursors.

This year, I spent six months in the Galvanize Full Stack program, and I gained a lot of exerience with JavaScript and React. At the end of the program,
Facebook released Relay. I really didn't have time to get into it since I needed to get a project done, but I found the idea of using a query language
interface between client and server really interesting. After graduation, I restarted a project that had been put aside when I was in school. I also wanted
to refresh my clojure skills after six months of mainly JavaScript. Well, I found Dave Nolan's alphas of
[om.next](https://github.com/omcljs/om/wiki/Quick-Start-(om.next)), and started playing with it.

Here is a video showing this simple application.

<iframe width="420" height="315" src="https://www.youtube.com/embed/T9X-MPiMBK0" frameborder="0" allowfullscreen></iframe>

Om.next is very different from om, and I think a lot better. Having worked with React, immutable.js, and the flux pattern, I found the om.next overall
design very comfortable. When working with the state in an atom stored locally in the client, it is like working with React and immutable using a really
good Flux library, and without the friction of using immutable data structures as a library in a language designed for mutability that
I experienced when using immutable.js.

I found om.next to be stable, and it has a reasonable amount of documentation. I did find that the documentation lacked details in some important
areas, particularly in the actual content of the arguments passed to the life cycle methods.

On to the code, which can be found on [github](https://github.com/jdunruh/condo-calendar).

    (def reconciler
      (om/reconciler
         {:state  (atom init-data)
           :parser (om/parser {:read read :mutate mutate})}))

    (defui Month
       static om/IQuery
         (query [this]
            [:month/weeks {:month/month-id (om/get-query Month-header)}])
       Object
         (render [this]
            (dom/div #js {:className "month"}
               (month-header (:month/month-id (om/props this)))
               (month-body (om/props this)))))

    (om/add-root! reconciler
         Month (gdom/getElement "app"))

This code defines a reconciler that has state stored in an atom (init-data) and a parser that uses the read and mutate multi-methods
to access the state data. The reconciler will call the function attached to the :read key in the parser to retrieve state data, and
the function called to :mutate key to update the state. The use of multi-methods is not required, but it can be cleaner that using
conditionals inside the read and mutate functions to determine what to do. The om/add-root! function attaches the Month component to the app element
in the DOM as the root of the tree and makes "reconciler" defined above the reconciler for this root.

Defui Month defines the month component. om/Iquery defines the query associated with this component. Om.next uses datomic pull syntax for the
queries. Facebook's Relay uses the same idea with graphQL for React. In this case, Month is the root element, so it gets the data it needs
directly (:month/weeks) and then gets the data needed for child components using om/get-query. Om.next will arrange to execute read functions for
each entry in the vector of om/IQuery of the root element, and pass the result as props to the reder lifecycle method. Note that the read
function must return a denormalized version of the application state suitable for rendering, and the root element passes the necessary data the
child element through props, just like a normal React application. You can the render method passes the month ID received in props to the
month-header, and all of props to the month-body.

Data normalization is an important concept in om.next. Om.next wants you to store data in a single store. That store might be an atom as in this
example, a local in-memory database like datascript, or a database on a remote server. Conceptually, there is a single source of truth for the
UI and it is the application state (or store in Flux terminology). A piece of data may be displayed to the user in multiple places, but if the data is
being updated, it is usually good to have it appear only once in the application state so that updates don't need to find multiple instances of the data
state and update each of them. With an in-memory database, this isn't a problem because the database takes care of storage and update, but when using
a simple atom, you must do it yourself.

So, let's take a look at my application satate

    (def init-data
      (let [today (t/today)
            month (t/month today)
            year (t/year today)]
            {:person/by-id   {0 {:color "red" :name "Judy" :id 0}
                              1 {:color "blue" :name "John" :id 1}
                              2 {:color "green" :name "Jake" :id 2}
                              3 {:color "yellow" :name "Susan" :id 3}}
             :month/month-id (t/date-time year month 1)
             :month          (six-weeks-containing-month year month)
             :days/by-date   {}
             :current-user   2}))

The application data (saved in an atom) is represented as a clojure map and is normalized. First, I compute he current month and year using
the cljs-time library aliased as t. Then I define the legal users under the :person/by-id key. This is another hash map keyd by the actual ID
of the users. By setting it up this way, read and mutate functions can access the user data with (get-in state [:person/by-id 0]) where 0
is he id for the requested user. The  first of the current month is in the :month/month-id
key. :month/month-id holds the currently displayed month on the calendar, it is just initialized to the current month. Because of the business
rules around this calendar, I chose to represent the month as six weeks, with the first of the month in the first of the weeks. Therefore,
I initialized the :month key as a list of six lists, each containing seven days. The :days/by-date key starts as an empty hash map. It gets
an entry for each assigned day in the calendar as they are assigned by the user. This is a map keyed by the day and containing the person in
the format [:person/by-id 0]. By using this format, the "join" of days with the person to which it is assigned becomes trivial. For each day,
get the associated value, which is a vector suitable for use as the second argument to get-in. Finally, the current user is stored under the
current-user key.

Now, a bit about read and mutate. In the end, read needs to return data in the format needed to render the component tree. Usually, this will be
denormalized. In this case, I have two read functions, one for the month-id used in the header, and one for the actual month used to rende the
calendar. Rather than use conditionals explicitly, I used polymorhism with multi-methods. Here is the code.

    (defn date-to-assignment [state date]
      (get-in state (get-in state [:days/by-date date] [nil]) {:name "available" :color "white"}))

    (defn denormalize-week [state week]
      (map #(merge % (date-to-assignment state (:days/by-date %))) week))


    (defn denormalize-month [state]
      (map #(denormalize-week state %) (:month state)))

    (defmulti read om/dispatch)

    (defmethod read :month/month-id
      [{:keys [state selector] :as env} key {:keys [month]}]
        (let [st @state]
            {:value (:month/month-id st)}))

    (defmethod read :month/weeks
      [{:keys [state selector] :as env} key]
        (let [st @state]
            {:value (denormalize-month st)}))

An om.next parser read function can take three parameters, the environment (env here), the key, and the params. I didn't use params in this
application, but it is intended to paramaterize a query, for example to specify start and end for a collection. In the read functions, I pull
the state and selector out using destructurinng. The state is the actual application state (here an atom), and the selectors specify what
to read from the state. When using the om/dispatch function to dispatch multi-methods, it uses key for the dispatch. If I chose to use a
monolithic function, I would be deciding what read to do by a conditional using key. The first method reads the month-id. The function returs
a map with the :value key set to the data. Return values from read need to be structured in this way. Since the month-id is simply a date-time,
this is a simple function. Getting the month (key :month/weeks) is more involved as I need to denormalize the data. To denormalize the data, I
call denormalize-month to map over the six weeks stored in the :month key under the application state returning a list of denormalized weeks.
The denormalize-weeks function maps across the list of seven days in a week, returning a list of seven days. It finds the assignment (to a person)
for the day, and merges the resulting map into the map that represents the day, accomplishing the denormalization. Date-to-assignment is a
fairly straightforward function as long as you know two things - get-in takes a third argument which is the default, or not found, value, and using
[nil] in the second argument of get-in will always return nil. So, for each day, I use get-in to find the day assignment from the :day/by-date
map. If there isn't an entry, I return [nil] as the default value. If it is found, the result will be [:person/by-id key]. I pass this result
as the second argument to get-in, to find the person data. If it isn't found , I return the default {:name "available" :color "white"}. So given
a date key from the month, these functions return either the entry from :person/by-id who is assigned this day, or the default. The read call
ultimately returns a list of weeks, each containing a list of days. Each day has the date data augmented with the assigned person or the default
"available". When this is passed to the render function thourgh props, all the data will be correct for each component in the render tree.

The ONLY way the app state is updated in an om.next application is a mutate function. The mutate functions can be triggered by user clicks as
seen here, or from data received by a remote. Here are my mutate functions.

    (defn add-assignment-to-calendar [state date assignee]
      (update state :days/by-date assoc date [:person/by-id assignee]))


    (defn remove-date-from-days [days date]
      (dissoc days date))

    (defn release-day [state date]
      (update state :days/by-date remove-date-from-days date))

    (defn next-month [current-month-start]
      "given a date-time, generate the date-time one month later"
        (t/plus current-month-start (t/months 1)))
    (defn last-month [current-month-start]
      "given a date-time, generate the date-time one month earlier"
        (t/minus current-month-start (t/months 1)))


    (defmulti mutate om/dispatch)

    (defmethod mutate 'day/change-state
      [{:keys [state]} _ {:keys [date] :as params}]
        (let [st @state]
          (if-let [day-assignee (get-in st [:days/by-date (str date)])]
             (do
                (if (= (second day-assignee) (:current-user st))  ; the date is assigned
                   {:value {:days/by-date date}                    ; it is assigned to the current user - so release
                    :action
                       (fn []
                          (swap! state release-day (str date)))}
                   {:value {:error "Cannot release someone else's day"}})) ; the date is assigned to someone else - user error
                      (do  {:value {:days/by-date date} ; the date is not assigned - so assign
                    :action
                       (fn []
                          (swap! state add-assignment-to-calendar (str date) (:current-user st)))}))))

    (defmethod mutate 'month/next
      [{:keys [state]} _ _]
        (let [st @state
              new-month (next-month (:month/month-id st))
              month (t/month new-month)
              year (t/year new-month)]
                {:value  {:month/month-id (date-key new-month)}
                 :action (fn []
                             (swap! state assoc :month/month-id new-month :month (six-weeks-containing-month year month)))}))

    (defmethod mutate 'month/previous
      [{:keys [state]} _ _]
        (let [st @state
                new-month (last-month (:month/month-id st))
                month (t/month new-month)
                year (t/year new-month)]
                  {:value  {:month/month-id (date-key new-month)}
                  :action (fn []
                              (swap! state assoc :month/month-id new-month :month (six-weeks-containing-month year month)))}))

The mutation functions take the same arguments as the read functions, but any data passed in will come in params. Mutation functions
return a value containing the read needed to update the UI in :value and the function to execute to actually update the state in
:action. After the mutate function returns, om.next will call the function in :action to update the app state.
The day/change-state mutation function looks at the current day assignment and classifies it in three ways. It may be unassigned. If so,
it is assigned to the current user. It may be assigned to another user, resulting in an error return (and no :action key in the returned
map), or it may be assigned to the current user, in which case it is released. The actual change to the app state atom happens using swap!.

the month/previous and month/next mutations are almost the same. These functions find the first of the previous (or next) month and and set
the action to generate the six weeks of the previous (or next) month and update the app state item :month key to the new month.

Since the calendar truly doesn't change, I chose not to store the calendar in the database. I just generate it as needed. I do, however need to
persist the assignments. Since this version of the project doesn't use a server, I just hold onto assignments in the app state when the month
changes. If the month changes back. the assignments will still be there.

This leads to an interesting situation for testing. With React and om.next, the generated DOM is a pure function of the app state. Therefore a lot
of testing can be done simply by using read and mutate functions without ever rendering a page. In fact, the user can be simulated as a series
of mutate calls since the only way for user interactions to change anything is to run a mutate function. Therefore a set of traditional tests using
cljs.tests of a series of property-based tests using test.check can test the updates to the app-state without resorting to clicking at a browser
or using selenium. Of course, there is still a need to have a human check the application to ensure that the colors and postions are correct and that the
user interactions actually work.

The last thing is the rendering. The render function comes in the Object part of the om.next protocal (where other React lifecycle methods also go).
This is mostly straigt forward.
\\#+BEGIN<sub>SRC</sub> clojure
(defui Day
  static om/Ident
  (ident [this {:keys [day/by-date]}]
       [:days/by-date by-date])
  Object
  (render [this]
    (let [day (:date (om/props this))
          color (:color (om/props this))
          name (:name (om/props this))]
          (dom/div #js{:className (str "day " color)

           :onMouseDown
               (fn [e]
                  (om/transact! reconciler \`[(day/change-state {:date ~(date-key day)})]))}
(dom/div #js {:className "day-no"} (t/day day))
(dom/span #js {:className "day-name"} name)))))

(def day (om/factory Day {:keyfn :days/by-date}))

(defui Week
  Object
  (render [this]
     (apply dom/div #js {:className "week"}
         (map day (om/props this)))))

(def week (om/factory Week {:keyfn #(:days/by-date (first %))}))

(defui Month-header
   static om/IQuery
  (query [this]
     '[:month/month-id])
   static om/Ident
   (ident [this props]
       [:month/month-id props])
   Object
   (render [this]
      (let [month-id (om/props this)]
         (dom/div #js {:className "month-header"}
            (dom/button #js {:className "change-month"

                 :onClick
                    (fn [e] (om/transact! reconciler '[(month/previous)]))}
   (dom/i #js {:className "fa fa-chevron-left fa-2x"}))
(dom/span #js {:className "month-year"}
   (cf/unparse (cf/formatter "MMMM YYYY") (t/date-time month-id)))
(dom/button #js {:className "change-month"

              :onClick
                 (fn [e] (om/transact! reconciler '[(month/next)]))}
(dom/i #js {:className "fa fa-chevron-right fa-2x"}))))))

(def month-header (om/factory Month-header))

\#+END<sub>SRC</sub>                                                                                                                                                    Here, the interesting parts are the use of om/factory, which creates a factory function needed to actually instantiate the compontents, and
the contents of the #js reader macro. #js allows the introduction of javascript into your clojurescript program. Here the classNames are inserted
and the click handlers are set up. Note that each click handler calls the mutate functions from above using om/transact!. This creates transactions
to the application state. If you bring up the console when running the applicaiton program, you can see the transaction IDs in the console each
time you transact a change into the application state.

That is pretty much it for this version of the application. Still to come: datascript integration and server interaction.

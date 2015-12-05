App Engine application for the Udacity Nanodegree: Full Stack Developer (Project 4)

## Products
- App Engine

## Language
- Python

## APIs
- Google Cloud Endpoints
- API Explorer

## Setup Instructions
1. Update the value of `application` in `app.yaml` to the app ID you
   have registered in the App Engine admin console.
1. Update the values in the `settings.py` to
   reflect the respective client IDs you have registered in the
   Developer Console.
1. Update the value of CLIENT_ID in `static/js/app.js` to the Web client ID

1. Run the app with the devserver using `dev_appserver.py DIR`, and ensure it's running by visiting your local server's address (by default localhost:8080.)
1. (Optional) Generate your client library(ies) with the endpoints tool.
1. Deploy your application.

## Tasks

## Task 1: Add Sessions to a Conference

### Define the following Endpoints methods

- `getConferenceSessions(websafeConferenceKey)`
   Given a conference, return all sessions

- `getConferenceSessionsByType(websafeConferenceKey, typeOfSession)`
   Given a conference, return all sessions of a specified type (eg lecture, keynote, workshop)

- `getSessionsBySpeaker(speaker)`
   Given a speaker, return all sessions given by this particular speaker, across all conferences

- `createSession(SessionForm, websafeConferenceKey)`
   Open only to the organizer of the conference

### Define Session class and SessionForm

In the SessionForm pass in:
- name
- highlights
- speaker
- duration
- typeOfSession
- date
- start time (in 24 hour notation so it can be ordered)

### Explain your design choices

Note: All endpoints can be assessed from the API Explorer

For session class, the name and speaker fields are of type stringProperty, date is a dateProperty, start time is a timeProperty and the  duration field is in minutes as an Integer property. The duration represents a maximum amount of minutes for which a user can search for sessions. All sessions less than or equal to the given duration will be returned.

Speaker is stored as a string property to allow for simple queries although at the moment this limits the amount of information which can be stored about a speaker. A stand alone entity can be created to allow for a more dynamic use of the object with additional information which can be used for more queries but for now to simplify the design, only the name is required hence the decision to store it as string. 

Sessions have been  implemented in a similar way to conferences and they are a child object of conference. A session form is used to copy data and return form objects. A user can save SessionsKeys to their profile as a wish-list for sessions they wish to attend.  There is validation to make sure no repeated values are saved in wish-list and that only urlSafe strings can be saved to the profile.

The featured speaker is being stored as a string in memcache once a particular speaker has more than one session for simplicity.


## Task 2: Add Sessions to User Wishlist

### Define the following Endpoints methods

- `addSessionToWishlist(SessionKey)`
   adds the session to the user's list of sessions they are interested in attending

- `getSessionsInWishlist()`
   query for all the sessions in a conference that the user is interested in

- `deleteSessionInWishlist(SessionKey)`
   removes the session from the user’s list of sessions they are interested in attending



## Task 3: Work on indexes and queries

### Part 1: Come up with 2 additional queries

Think about other types of queries that would be useful for this application.
Describe the purpose of 2 new queries and write the code that would perform them.


#### query 1

Get sessions by their duration in minutes. All sessions less than or equal to the given duration in minutes will be returned

```py
@endpoints.method(SessionsByDurationForm, SessionForms,
            http_method='GET',
            name='getSessionsByDuration')
    def getSessionsByDuration(self, request):
        """Return sessions by their duration."""
        # get sessions less than or equal to the given time in minutes
        sessions = Session.query(Session.duration <= int(request.duration))

        return SessionForms(items=[self._copySessionToForm(session) for session in sessions])

```

#### query 2
Get all sessions for all conferences which are within a given location (city).

```py
@endpoints.method(SESSION_GET_REQUEST_BY_LOCATION,
                      SessionForms,
                      http_method='GET',
                      name='getSessionsByLocation')
    def getSessionsByLocation(self, request):
        """Return all conference sessions within a given city """
        confs = Conference.query(Conference.city==request.city)
        sessions = []
        for conf in confs:
            sessions += Session.query(ancestor=conf.key)

        return SessionForms(items=[self._copySessionToForm(session) for session in sessions])

```

### Part 2: Solve the following query related problem

> Let’s say that you don't like workshops and you don't like sessions after 7 pm.
> How would you handle a query for all non-workshop sessions before 7 pm?

#### What is the problem for implementing this query?
An inequality filter can be applied to at most one property. This problem requires an inequality filter on two properties.

#### What ways to solve it did you think of?

#### Solution:

Assuming that we have a list of session types ( lets call it SessionTypes), we can combine the two inequalities to come up with a solution by first filtering out the type which is workshop('WORKSHOP' in this case) (this solution excludes sessions which begin before 7PM and run past it. That would require an extra condition):

```py
session_types = [type for type in SessionTypes if type != 'WORKSHOP']
time = datetime.strptime("19:00", "%H:%M").time()

sessions = Session.query(ndb.AND(Session.typeOfSession.IN(session_types), Session.startTime < time))

return sessions
```
## Task 4: Add a Task
When a new session is added to a conference, check the speaker. If there is more than one session by this speaker at this conference, also add a new Memcache entry that features the speaker and session names. You can choose the Memcache key.
The logic above should be handled using App Engine's Task Queue.

### Define the following Endpoints method

- `getFeaturedSpeaker()`

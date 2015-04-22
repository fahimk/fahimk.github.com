---
layout: post
title: "RxJava with MVP to handle configuration changes"
description: ""
category: 
tags: []
---
{% include JB/setup %}
###Intro
Configuration changes with long running tasks (network calls, database operations) often result in a bad user experience. Here is a simple app that gets a list of artists based on a search. Even though the network call is maintained, through the orientation, the progress circle disappears. There is no indication that the results are still coming in, but they do eventually. 

![array items and state not saved]({{ site.url }}/assets/bad2.gif)

If the callback isn't persistant through the lifcycle changes, then you could end up with just a blank search page on every rotation.

Since the activity is being recreated on the orientation change, we need to keep track of both the network data that is coming in and the current "state" of the activity. State in this case refers to what is currently being displayed to the user (the progress circle, list of results, or empty search page)
<br />
<br />

###Options
Up until now, I've used robospice which gave the ability to get pending listeners after a rotation. However, it doesn't take care of saving the view state so that needs to be handled independently

I recently stumbled upon two frameworks that take this issue on directly:

<a href="https://github.com/doridori/Dynamo">https://github.com/doridori/Dynamo</a>

<a href="https://github.com/sockeqwe/mosby">ttps://github.com/sockeqwe/mosby</a>

Both have a similar style, where there is a component independent of the activity life cycle that performs the long running task, and then reports back to the activity/view/fragment (while Mosby does offer the ability to tie the Presenter to the Activity lifecycle by cancelling the long running task and then restoring it, I really don't like that solution). They also have a way to keep track of states with built in options for common states: loading, content, error.

I think both libraries are great, the Mosby blog post by Hannes (<a href="http://hannesdorfmann.com/android/mosby/">http://hannesdorfmann.com/android/mosby/</a>) had examples with explanations which made it easy to follow. 


###Mosby
Mosby follows the Model View Presenter programming design pattern. Checkout Hannes' article for more specifics on how MVP is intended to work. 
![mvp diagram]({{ site.url }}/assets/diagram.png)
<br />
<br />

####View
in the view we need to define the methods that the presenter will call based on what needs to be shown
{% highlight java %}
    public void setData(List<Artist> artists) {
        searchListAdapter.setArtists(artists);
        searchListAdapter.notifyDataSetChanged();
	}

    
    public void showLoading() {
        getViewState().setStateShowLoading();
        viewFlipper.setDisplayedChild(VIEWFLIPPER_LOADING);
    }

    public void showSearchList() {
        getViewState().setStateShowSearchList();
        viewFlipper.setDisplayedChild(VIEWFLIPPER_RESULTS);
    }
{% endhighlight %}

Notice how the ViewState is updated based on what the presenter has asked us to display. That way when we have a configuration change, we can save the ViewState to a bundle, and retrieve it in OnCreate if needed.
<br />
<br />

####ViewState
{% highlight java %}
{
    private final int STATE_SHOWING_SEARCH_LIST = 0;
    private final int STATE_SHOWING_LOADING = 1;

    private int currentState = 0;
}
{% endhighlight %}
<br />

####Presenter
when the user presses search, we ask the presenter to perform a search with a query. here is what that method looks like

{% highlight java %}
    @GET("/search?type=artist")
    Observable<ArtistSearchResponse> artistSearch(
            @Query("q") String query);

    ...
    ...

    public void searchForArtists(String query) {
        if (isViewAttached()) {
            getView().showLoading();
        }

        spotifyService.artistSearch(query)
                .delay(5, TimeUnit.SECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<ArtistSearchResponse>() {
                    @Override
                    public void onCompleted() {
                        if (isViewAttached()) {
                            getView().showSearchList();
                        }
                    }

                    @Override
                    public void onError(Throwable e) {
                        if (isViewAttached()) {
                            getView().showError(e);
                        }
                    }

                    @Override
                    public void onNext(ArtistSearchResponse artistSearchResponse) {
                        if (isViewAttached()) {
                            getView().setData(artistSearchResponse.artists.items);
                        }
                    }
                });
    }
{% endhighlight %}

It subscribes to the Observable provided by Retrofit and lets the View know what to display. The great part about Mosby is that it gives you classes to extend that contain a lot of the boilerplate code. The configuration I prefer the most is:

View extends MvpViewStateFragment
Presenter extends MvpBasePresenter

There is an optional Rx module, which currently only supports Lce (loading, content, error) which is a specific configuration where you have an R.id.contentView, R.id.errorView, and an R.id.loadingView where Mosby will show/hide the appropriate one. It's nice if you need to put together a quick app, but there isn't much flexibility if you want to display the error as a toast for example.

Although I'm using RxJava here, the method searchForArtists can use a regular Retrofit call and update the View in the callbacks.
<br />
<br />

####Maintaining the presenter
In the SpotifyArtists example I've used a retained fragment to hold the ViewState and the adapter for the RecyclerView. 

Mosby does have the ability to use an Activity or regular Fragment, however then you are cancelling the requests on orientation change, and then re-requesting after the new activity is created. This would work in some cases, but what if you had a registration form, and the server is still creating the account when you rotate the device. You could cancel and retry that but it might fail because the username now exists.

The other option Hannes suggested is using a Singleton Presenter that would be injected via Dagger instead of creating a new presenter. I think this is a good alternative if you are trying to stay away from fragments.
<br />
<br />

####Final result
Here is the code: <a href="https://github.com/fahimk/SpotifyArtists">https://github.com/fahimk/SpotifyArtists</a>

![final result]({{ site.url }}/assets/good.gif)
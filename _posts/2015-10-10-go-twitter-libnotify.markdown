---
layout: post
title: Go + twitter + libnotify
---

A [Go] egy viszonylag új nyelv a Google-től. Típusos, egy darab binárisba fordított (azaz piszok gyors) és nyelvi szinten támogatja a párhuzamosságot. Szemeztem vele egy ideje, aztán nemrég kipróbáltam.
Az első program, amit írtam benne egy twitter értesítő volt, mert miért ne? Két elég egyszerű dolog kombinációja, vannak rájuk kész libek, és életszagú.

Választott twitter libem az [anaconda], libnotify-ra pedig a [go-notify](https://github.com/mqu/go-notify/). A projekt mappa létrehozása után gyorsan `go get`eltem is őket; a twitter fejlesztői oldalán pedig regisztráltam egy új appot, szerezve négy tokent. Ezeket env változóban passzolom át a kódnak (mert sehogy nem sikerült json-ból beolvasnom, azóta leesett, hogy azért, mert kisbetűs névvel voltak a structomban), így:

{% highlight go %}
consumerKey := os.Getenv("TWITTER_CONSUMER_KEY")
consumerSecret := os.Getenv("TWITTER_CONSUMER_SECRET")
accessToken := os.Getenv("TWITTER_ACCESS_TOKEN")
accessTokenSecret := os.Getenv("TWITTER_ACCESS_TOKEN_SECRET")
{% endhighlight %}

Amivel aztán initelhetjük az anacondát, és rögtön próbáljuk is ki, minden stimmel-e:

{% highlight go %}
anaconda.SetConsumerKey(consumerKey)
anaconda.SetConsumerSecret(consumerSecret)
api := anaconda.NewTwitterApi(accessToken, accessTokenSecret)

user, err := api.GetSelf(nil)
if err != nil {
	panic(err)
}

fmt.Printf("Logged in as @%s\n", user.ScreenName)
go doNotify(
	"Twitter",
	"Logged in as @" + user.ScreenName,
	user.ProfileImageURL)
{% endhighlight %}

A notificationöket megjelenítő kód külön függvénybe került már csak azért is, mert egy bug miatt minden notif előtt initelni, utána pedig uninitelni kell a libraryt - a kettő között pár pásodperc késleltetéssel. Extraként bekerült a user képének letöltése, rengeteg hibakezeléssel :)

{% highlight go %}
func doNotify(title, text, image string) {
	if "" != image {
		filename := "/tmp/twitter-" + getMd5(image)
		output, _ := os.Create(filename)
		response, _ := http.Get(image)
		io.Copy(output, response.Body)

		defer output.Close()
		defer response.Body.Close()
		defer os.Remove(filename)

		image = filename
	}

	notify.Init("twitter-stream-notify")
	defer notify.UnInit()

	notification := notify.NotificationNew(title, text, image)
	notify.NotificationShow(notification)

	time.Sleep(10 * time.Second)
	notify.NotificationClose(notification)
}
{% endhighlight %}

Mivel `go` prefixszel hívom mindenütt, külön szálon indul, a sleep véletlen sem fogja meg a fő threadet.

Szóval már bejelentkeztünk, megvan az első notifünk is, még hátravan a tweetekre várás és azokról értesítés összerakása. Ezzel a résszel szórakoztam a legtöbbet, az anaconda dokumentációja nem épp részletes a streamekkel kapcsolatban, úgyhogy a kódból szedtem össze az infot - nem volt egyszerű egy tökúj nyelven.

Először a twittertől kell elkérnünk a user streamjét - ez azokat a tweeteket jelenti valós időben, amiket az általa követettek írnak. A lib elfedi ennek a részleteit, és a végén egy go channelbe pakolja nekünk, ebből kell végtelenciklusban olvasnunk. Ide jöhet sokfajta üzenet, minket ezek közül csak a tweetek érdekelnek - a típus eldöntése a goban elsőre nem egyértelmű, de utólag nagyon kellemes és praktikus megoldás.

{% highlight go %}
stream := api.UserStream(nil)

for {
	select {
	case data := <-stream.C:
		if tweet, ok := data.(anaconda.Tweet); ok {
			fmt.Printf("@%s: %s\n", tweet.User.ScreenName, tweet.Text)
			go doNotify("@" + tweet.User.ScreenName,
				tweet.Text,
				tweet.User.ProfileImageURL)
		}
	}
}
{% endhighlight %}

A teljes kód fenn van [githubon]. Az egészet kb. egy délután alatt raktam össze, és nagyon megtetszett a nyelv. Azóta nekiestem mégegy projektnek, az melóhoz kapcsolódik, egy nodejs microservice-t kezdtem újrairni goban, egy hétvége alatt összeálltak a fő funkciók, azonos körülmények között hatszor gyorsabb (`ab`-val mérve) és jelentősen kevesebb ramot eszik. Biztosan fogok még ezzel a nyelvvel játszani.

 [go]: https://golang.org/
 [anaconda]: https://github.com/ChimeraCoder/anaconda
 [gonotify]: https://golang.org/
 [githubon]: https://github.com/maerlyn/go-twitter-libnotify


---
layout: post
title:  "Why Spotify's domain is too complicated and how to fix it"
date:   2017-11-21 16:25:00 +0200
published: true
categories: ['spotify', 'software design', 'domain modeling']
---
Spotify is that really popular music streaming service most of you probably know
about. If you're familiar with the app you might also know that they give you
lots of concepts to wrap your head around:

- Playlist (A list of songs)
- Albums (A list of songs corresponding to a record an artist has put out)
- Artists (A list of albums the artist has worked on + a biography + related artists)
- Folders (Can contain playlists and more folders)
- Following (You can follow artists, and playlists alike)
- Favorites (Keeps track of your favorite songs)

These concepts are all useful in their own right but they do not work together
very well. For example if I make a playlist of an album Spotify forgets the fact
that it is an album altogether. Which means that when I play a song off the
album page it's not the same song as the song on my playlist of the album
because the playlist is not the album. Confused? So am I.

I propose a unification of all these concepts by making everything a **playlist**:

An **album** becomes a playlist and can be used as such. A folder is a set of
playlists but also a playlist comprised of this set. This allows you to play
music from the folder as a playlist when you want to, without losing the
structure of all the separate playlists in the folder.

**Artists** also become playlists, where the playlist is updated every time the
artist puts out a new record.

Want to keep track of your favorites? Simply create a playlist called
**favorites** and put your favorite songs in there. No need for a separate
'favorites' concept.

**Following** is still very useful, allowing you refer to playlists by another
artist, user, whatever and receiving updates to the playlist as well.

# Set theory

If we look at set theory we can see many parallels with this new approach to
playlists:

- A folder is a **union** of the playlists inside it.
- An artist's playlist is the **union** of all the album playlists.
- A playlist containing all songs off an album is a **superset** of the album's
  playlist.
- A playlist containing some (but not all) songs off an album is a **subset** of
  the album's playlist.

Looking at it this way might not be immediately practical but it does show the
compositionality of the playlist concept very well.

# Conclusion

Designing anything, including software, should trigger us to think about how the
different elements of the design complement each other. A beautiful design is
often one comprised of a few parts that work well together. If the elements do
not compose with one another a design can often feel like it has been put
together in an ad-hoc fashion.

As with the Spotify app, we see that all these concepts like album, artist,
playlist almost co-incidentally exist in the same space and are very weakly
related. As if each concept was designed by someone else without thinking about
the whole.

As you can see a unification and broad application of the playlist concepts
neatly links everything together. The user only really needs to understand the
concept of a playlist and all other concepts logically tie into that concept.

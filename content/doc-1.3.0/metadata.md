Customize metadata using Liquidsoap
===================================
Liquidsoap has several mechanism for manipulating the metadata attached to your
stream. In this page we quickly detail and compare the different operators, see
the [language reference](reference.html) for full details about them.

**Warning**. The protocol used by Shoutcast and Icecast before version 2 does
not support many fields. It mainly support one: `song`. So, if you
need to customize the metadata displayed by these servers, you should
customize only the `song` metadata.

The annotate protocol
---------------------
The metadata are read from files, so the most simple way is to properly tag the
files. However, if it not possible to modify the files for some reason, the
`annotate` protocol can be used in playlists to insert and modify some
metadata. For instance, in the playlist

```

annotate:title="Title 1",artist="Artist 1":music1.mp3
annotate:title="Title 2",artist="Artist 2":music2.mp3
```

the title metadata for file music1.mp3 will be overridden and changed to ``Title
1'' (and similarly for the artist).

Rewrite metadata
----------------
`rewrite_metadata` rewrites metadata using a list of
(target,rules). The semantic for the replacement rules is that of the
`%` function. Namely, `(pattern % [...,(k,v),...])`
changes in `pattern` occurences of:

* `'$(k)'` into `"v"`;
* `'$(if $(k2),"a","b")'` into `"a"` if `k2`

is found in the list, `"b"` otherwise.

A sample code using this operator can be:

```
# The 'display_artist' field is passed using annotate.
pattern =
  '$(if $(display_artist),"$(display_artist)","$(artist)")'
rewrite_metadata([("artist",pattern)],source)
```

Map metadata
------------
The `map_metadata` operator applies a specified function to transform
each metadata chunk of a stream. It can be used to add or decorate metadata, but
is also useful in more complex cases.

A simple example using it:

```
# A function applied to each metadata chunk
def append_title(m) =
  # Grab the current title
  title = m["title"]

  # Return a new title metadata
  [("title","#{title} - www.station.com")]
end

#??Apply map_metadata to s using append_title
s = map_metadata(append_title, s)
```

The effect of `map_metadata` by default is to update the metadata with the
returned values. Hence in the function `append_title` defined in the code above
returns a new metadata for the label `title` and the other metadata remain
untouched. You can change this by using the `update` option, and you can also
remove any metadata (even empty one) using the `strip` option.

See the documentation on `map_metadata` for more details.

A more complex example is the `rewrite_metadata` operator, which is
implemented using `map_metadata` as follows:

```
# Rewrite metadata on the fly using a list of (target,rule).
# @category Source / Track Processing
# @param l  List of (target,value) rewriting rules
# @param ~insert_missing \
#           Treat track beginnings without metadata \
#           as having empty ones.
def rewrite_metadata(l,~insert_missing=true,s)
  def map(m)
    def apply(x)
      label = fst(x)
      meta = snd(x)
      if list.mem_assoc(label,l) then
        pattern = l[label]
        (label,pattern % m)
      else
        (label,meta)
      end
    end
    m = list.map(apply,m)
    def add(m,x)
      label = fst(x)
      pattern = snd(x)
      # If m does not have (label,_), then it was
      # not processed previously, we have to
      # add it now..
      if not list.mem_assoc(label,m) then
        list.append(m,[(label,pattern % m)])
      else
        m
      end
    end
    list.fold(add,m,l)
  end
  map_metadata(map,insert_missing=insert_missing,s)
end
```

Insert metadata
---------------
### Using the telnet server
This operator is used for inserting metadata using a server command. If you have
an `server.insert_metadata` node named `ID` in your configuration, as in

```
server.insert_metadata(id="ID", source)
```

you can connect to the server (either telnet or socket) and execute commands
like

```
ID.insert key1="val1",key2="val2",...```

### In Liquidsoap
Sometimes it is desirable to change the metadata dynamically when an event
occurs. In this case, the function `insert_metadata` (not to be confused with
`server.insert_metadata` above) can be used: when applied to a source it returns
a pair constituted of a function to update metadata and the source with inserted
metadata.

For instance, suppose that you want to insert metadata on the stream using the
OSC protocol. When a pair of strings ``title'' ``The new title'' is recieved on
`/metadata`, we want to change the title of the stream accordingly. This can be
achieved as follows.

```
# Our main music source
s = playlist("...")
s = mksafe(s)

# Create a function to insert metadata
ms = insert_metadata(s)
# The function to insert metadata
imeta = fst(ms)
# The source with inserted metadata
s = snd(ms)

# Handler for OSC events (gets pairs of strings)
def on_meta(m) =
  # Extract the label
  label = fst(m)
  # Extract the value
  value = snd(m)
  # A debug message
  print("Insert metadata #{label} = #{value}")
  # Insert the metadata
  imeta([(label,value)])
end

# Call the above handler when we have a pair of strings on /metadata
osc.on_string_pair("/metadata",on_meta)

# Output on icecast
output.icecast(%mp3,mount="test.mp3",s)
```

We can then change the title of the stream by sending OSC messages, for instance

```
oscsend localhost 7777 "/metadata" ss "title" "The new title"```



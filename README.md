# Leaflet_map_v2
This is a readme for your new Budibase plugin.

# Description
A component which has the ability to add multiple layers, different icons, and more. Whether you just want to edit the default marker and color or do more advanced customization based on your data! It also allows you to have multiple Leaflet Layers!

##Basic Customization

**Icon**
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/default_icon.png)

**Color**
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/default_marker_color.png)

**Tile URLs**
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/layer_url.png)

##Advanced Customization

For advanced customization the following highlighted fields need to be set.
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/advanced_customization.png)

There are three different icon customizations. 
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/marker_icon_type.png)

The "Key Field" is the field which will be used to determine what icon/color/ or url to use when the row's data corresponds with the key value. You can define the key value pairs by clicking define options.

####Icon
The icon customization type uses [Remix Icon](https://remixicon.com). You can find the correct icon name by looking at this [link](https://remixicon.com) or utilizing Budibase's icon picker.
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/default_icon.png).
Be sure to spell the name correctly with the "ri" prefix.

Here's an example of it working.
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/marker_icon_edit_type.png).

####Color
The color customization type uses standard RGB HTML Codes. Be sure to prefix it with a "#". You may also use Budibase's spectrum colors as well var(--spectrum-global-color-static-green-400) in this format. The color advanced customization will use the default marker set 

Here's an example of it working.
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/marker_icon_color_edit.png).

####URL
The url customization type uses standard urls but be aware some URLs might cause issues due to CORS. Instead it's preferred to (host the images within budibase itself)[https://docs.budibase.com/docs/image]. You can also use base 64 images if that's easier for you!

Here's an example of it working.
![alt text](https://github.com/JayP718/Customizable_Leaflet_Map/blob/main/assets/marker_url_icon_edit.png).










Find out more about [Budibase](https://github.com/Budibase/budibase).



## Instructions

To build your new  plugin run the following in your Budibase CLI:
```
budi plugins --build
```

You can also re-build everytime you make a change to your plugin with the command:
```
budi plugins --watch
```


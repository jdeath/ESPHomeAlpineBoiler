# ESPHomeAlpineBoiler
Documentation for ESPHome Modbus Boiler Reader

This is documenation to add an Alpine Burnham (US Boiler) Boiler to homeassistant. Tested on model number: ALP150BW-4T02 but should work on most similar models.
Putting this togeather took research from many people online
1. https://github.com/alanmitchell/mini-monitor/blob/master/readers/sage_boiler.py
1. https://github.com/jdleslie/sage2-boiler/blob/master/sage_boiler.py
1. https://www.ccontrols.com/support/dp/Sage2.doc

Components Needed:
1. TTL to RS485 Module 485 . https://www.amazon.com/gp/product/B07YZTGHGG Could use the more common 485Max, but this board is a little easier as does not need flow control.(but may need a resister)
2. Wemos Mini D1. https://www.amazon.com/gp/product/B081PX9YFV . Could use any ESP module you like, but these are cheap
3. Cat 5/6 Ethernet Cable (which you will destroy)
4. 120 Ohm Resitor. (I used a 100 Ohm and still worked) This was the trick, without this I was beating my head against the wall!


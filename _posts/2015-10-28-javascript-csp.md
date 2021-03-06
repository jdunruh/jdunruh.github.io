---
layout: post
---

# JavaScript Communicating Sequential Processes<a id="sec-1" name="sec-1"></a>

In 1985, Tony Hoare published his book on [Communicating Sequential Processes](http://www.usingcsp.com/cspbook.pdf)
Communicating sequential processes started to gain momentum with the release of the Go programming language
with the inclusion of go-routines. The clojure community picked it up with the introduction of core.async.
There is a port of core.async to JavaScript - [js-csp](https://github.com/ubolonton/js-csp) ClojureScript programmers often use core.async in front
end applications to organize anynchrnous event handling. I wanted to explore CSP in JavaScript, so I created a
simple browser application that emulates (in a very basic way) a network management console.
Here is a demo video.

<iframe width="420" height="315" src="https://www.youtube.com/embed/pm8d_zdcf-A" frameborder="0" allowfullscreen></iframe>

Here's the code

## Network Element<a id="sec-1-1" name="sec-1-1"></a>

    // set the alarm status class on the element
    var setStatus = function(element,status) {
        $(element).removeClass('minorAlarm majorAlarm noAlarm').addClass(status);
    };

    // element is the selector for this network element
    var networkElement = function(errorChan, northBound, element) {
        var errors = 0;
        csp.go(function*() {
            while (true) {
                var timeout = csp.timeout(5000); // timeout in 5 seconds
                var result = yield csp.alts([errorChan, timeout]);
                if (result.channel === timeout) {
                    var status = (errors >= 5) ? 'majorAlarm' : 'noAlarm';
                    errors = 0;
                    setStatus(element, status);
                    yield csp.put(northBound, {element: element, status: status}); // send out of service indicator to the northbound interface
                } else { // input was from errorChan, which is an error indication
                    if(++errors === 5) {
                        setStatus(element, 'majorAlarm');
                        yield csp.put(northBound,{element: element, status: 'majorAlarm'});
                    }
                }
            }
        });
    };

Communicating sequential processes send and receive messages on channels, using put to send on a channel, get to
receive from a channel, and alts to receive from one of several channels. This function is the code that monitors
a network element and displays its status.
The network element gets a CSP channel on which it will get error indications, errorChan (in this example mouse clicks),
a channel for the northbound
interface, northBound - the interface to higher level parts of the system, and the html ID attribute
of the network element in the DOM. So, like a real network management app,
this code runs forever, waiting on some channel each time around the loop. Here, alts waits on two channels,
one that receives error
notifications and one that recieves a timeout after 5000 mSec. When an "error" (mouse click) occurs, the error handler
sends a message down errorChan. The networkElement manager simply counts them up, and if it has seen more than
five, it declares the element has a problem by setting the status to majorAlarm. In this app, jquery calls set the
class of the block to majorAlarm, turning it red. It also sends a notification to a higher level network management
entity on the northBound channel.

When the alts call returns the timeout channel, the network state is set appropriately and the error count is cleared,
implementing a crude sliding window protocol.

An instance of the networkElement communicating sequential process is instantiated for each element in the network -
one for the load balancer, one for each web server, and one for each database server.

Since js-csp is based on ES2015 generators, the body of the loop in inside a generator function (function\*) and
each call that could block the generator is preceeded by a yield keyword.

## Element Group<a id="sec-1-2" name="sec-1-2"></a>

The element group receives notifications on its southbound interface from the network elements, and sends notifications
on its northbound interface to higher levels - in this case the entire data center represented by the frame around all
the element groups. The same code handles both the element groups (web servers, etc) and the whole data center, since
the entire data center is simply a group of element groups. Here's the code.

    // set up alarm status on composite elements - groups of other elements
    var setGroupAlarm = function(element, numAlarms, majorThreshold, minorThreshold) {
        var status = 'noAlarm';
        if(numAlarms >= majorThreshold) {
            status = 'majorAlarm'
        } else if(numAlarms >= minorThreshold) {
            status = 'minorAlarm';
        } else
          status =  'noAlarm';
        setStatus(element, status);
        return {element: element, status: status};
    };

Starting the the setGroupAlarm function, we decide which alarm is appropriate by looking at the number of
alarms on the network elements and the threshold for major alarms (red) and minor alarms (yellow).

    // subTendingElements is an array of the selector strings for the subtending network elements on the page
    // groupSelecton is the selector for this group in the page
    var elementGroup = function (northBound, southBound, subTendingElements, minorThreshold, majorThreshold, groupSelector, timer) {
        csp.go(function*() {
            var numMajor = 0;
            var elements = {}; // keep track of each subtending element
            // use object as a hash to track reporting by subtending elements
            subTendingElements.forEach(function (el) {
                elements[el] = {status: "noAlarm", updated: true};
            });
            var timeout = csp.timeout(timer);

            while(true) {
                var result = yield csp.alts([southBound, timeout]);
                if(result.channel === timeout) {
                    // on timeout, update status to saved value and set saved value to majorAlarm so that it will come up as
                    // a major alarm if an element doesn't check in each minute, indicating the problem
                    for (var i in elements) {
                        if (elements.hasOwnProperty(i)) {
                            if (elements[i].updated === false && elements[i].status != 'majorAlarm') {
                                elements[i].status = 'majorAlarm';
                                ++numMajor;
                            }
                            elements[i].updated = false;
                        }
                    }
                    // update status and send status to northBound so we don't time out
                   yield csp.put(northBound, setGroupAlarm(groupSelector, numMajor, majorThreshold, minorThreshold));
                    timeout = csp.timeout(timer);
                } else { // not a timeout
                    if (result.value.status === 'majorAlarm') {
                        if (elements[result.value.element].status != 'majorAlarm')
                            yield csp.put(northBound, setGroupAlarm(groupSelector, ++numMajor, majorThreshold, minorThreshold));
                    } else {
                        if (elements[result.value.element].status === 'majorAlarm')
                            yield csp.put(northBound, setGroupAlarm(groupSelector, --numMajor, majorThreshold, minorThreshold));
                    }
                    elements[result.value.element].status = result.value.status;
                    elements[result.value.element].updated = true;
                }
            }
        })
    };

A elementGroup receives the northBound channel to report to the "data center" level and a southBound channel to recieve
reports from the lower level elements - either a group of networkElement(s), or other elementGroup(s). It also receives
an array of the ID attributes of the network elements it surveils (subTendingElements), the threshold for the number
of alarms necessary to declare a major or minor alarm for the elementGroup (minorThreshold and majorThreshold), the ID
attribute of the group (groupSelector), and the timeout value for the elementGroup.

First, we create an array of the subtending network elements (elements) and initialize each element with a status
and update flag, and we create a timer. Once this initialization is complete, we loop looking for timeouts and messages
from subtending elements on the southbound interface using alts. On a timeout, we loop through the network elements,
and we set a major alarm status if we haven't heard from the element as reflected in the updated property, and we count
the number of major alarms active. We then notify the process on the northbound interface of our status and reset the
timer.

If it isn't a timeout, it must be a message on the southbound interface. We set the alarm value as required based on the
incoming message.

Just as with the networkElement, there is one elementGroup instantiated for each group entity in the network.

Here is the code to wire it all up

    // wire up the dataflow

    // create the channels
    var webServerNorthbound = csp.chan();
    var loadBalancerNorthbound = csp.chan();
    var databaseServerNorthbound = csp.chan();
    var groupNorthbound = csp.chan();

    // wire the network elements
    var setNetworkElementHandlers = function(selector, northBound) {
        $(selector).each(function(index, el) {
            var ch = csp.chan();
            networkElement(ch, northBound, '#' + el.getAttribute('id'));
            $(el).click(function () {
                csp.putAsync(ch, 'Error');
            });
        });
    };

    setNetworkElementHandlers('.loadBalancer', loadBalancerNorthbound);
    setNetworkElementHandlers('.webServer', webServerNorthbound);
    setNetworkElementHandlers('.databaseServer', databaseServerNorthbound);

    // for the container, the NB goes into the bit bucket, so use a dropping buffer
    var containerNorthbound = csp.chan(csp.buffers.dropping(1));

    // wire the groups
    elementGroup(groupNorthbound, loadBalancerNorthbound, ['#LB1'], 1, 1, '#loadBalancers', 6000);
    elementGroup(groupNorthbound, webServerNorthbound, ['#WS1', '#WS2', '#WS3', '#WS4'], 1, 2, '#webServers', 7000);
    elementGroup(groupNorthbound, databaseServerNorthbound, ['#DS1', '#DS2'], 1, 1, "#databaseServers", 8000);

    // wire the container
    elementGroup(containerNorthbound, groupNorthbound, ['#loadBalancers', '#webServers', '#databaseServers'], 1, 1, '#container', 9000);

First we create all the channels, then we attach click handlers to all the network elements so we can simulate
errors. Note that the only function in the handler is to send an error indication to the networkElement process
for the simulated device (web server, etc) on its northbound interface. Then we put together the elementGroups. The
only other interesting part of this code is thecontainerNorthbound channel, which uses dropping buffers. The frame,
or container, uses the same code as any other elementGroup. If we had a default channel type, then sending on the
containerNorthbound channel would block the process forever since nothing is listening to the channel. By using a
dropping buffer, all these messages just go into the bit bucket and all the code can be used for both grouping levels.

I hope this gives some insight into using communicating sequential processes in JavaScript. The code can be found in
my [repository on Github](https://github.com/jdunruh/js-csp_demo)

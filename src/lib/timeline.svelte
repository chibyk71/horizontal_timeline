<script lang="ts">
import {
    onMount
} from "svelte";
import type {
    E
} from "./type.js";
import {
    Util
} from "./util.js";
import {
    writable
} from "svelte/store";
import {
    SwipeContent
} from "./swipe.js";

function initTimeline (dateValues: string | any[], date: NodeListOf < E > , minLapse: number) {
    // set dates left position
    let left = 0;
    for (let i = 0; i < dateValues.length; i++) {
        let j = (i == 0) ? 0 : i - 1;
        let distance = daydiff(dateValues[j], dateValues[i]),
            distanceNorm = (Math.round(distance / minLapse) + 2) * eventsMinDistance;
            // console.log(distance+" "+ minLapse);
        if (distanceNorm < eventsMinDistance) {
            distanceNorm = eventsMinDistance;
        } else if (distanceNorm > eventsMaxDistance) {
            distanceNorm = eventsMaxDistance;
        }
        left += distanceNorm;
        date[i].setAttribute('style', 'left:' + left + 'px');
        
    }
    
    
    // set line/filling line dimensions
    line.style.width = (left + eventsMinDistance) + 'px';
    // console.log(line);
    
    lineLength = left + eventsMinDistance;
    // reveal timeline
    Util.addClass(element, 'cd-h-timeline--loaded');
    selectNewDate(newDateIndex, date, oldDateIndex, $selectedDate);
    resetTimelinePosition('next');
};

function initEvents(navigation: NodeListOf < E > , date: NodeListOf < E > , content: NodeListOf < E > , newDateIndex: number, oldDateIndex: any) {
    // click on arrow navigation
    navigation[0].addEventListener('click', function(event) {
        event.preventDefault();
        translateTimeline(navigation, 'prev');
    });
    navigation[1].addEventListener('click', function(event) {
        event.preventDefault();
        translateTimeline(navigation, 'next');
    });

    //swipe on timeline
    new SwipeContent(datesContainer);
    datesContainer.addEventListener('swipeLeft', function(event: any) {
        translateTimeline(navigation, 'next');
    });
    datesContainer.addEventListener('swipeRight', function(event: any) {
        translateTimeline(navigation, 'prev');
    });

    //select a new event
    for (let i = 0; i < date.length; i++) {
        date[i].addEventListener('click', function(event: Event) {
            event.preventDefault();
            selectNewDate(newDateIndex, date, oldDateIndex, event.target);
        });

        content[i].addEventListener('animationend', function(event) {
            if (i == newDateIndex && newDateIndex != oldDateIndex) resetAnimation(newDateIndex, oldDateIndex);
        });
    }
};

function updateFilling() { // update fillingLine scale value
    let dateStyle = window.getComputedStyle($selectedDate!, null),
        left: number | string = dateStyle.getPropertyValue("left"),
        width = dateStyle.getPropertyValue("width");

    left = Number(left.replace('px', '')) + Number(width.replace('px', '')) / 2;
    fillingLine.style.transform = 'scaleX(' + (left / lineLength) + ')';
};

function translateTimeline(navigation: NodeListOf < E > , direction: string) { // translate timeline (and date elements)
    let containerWidth = datesContainer.offsetWidth;
    if (direction) {
        translate = (direction == 'next') ? translate - containerWidth + eventsMinDistance : translate + containerWidth - eventsMinDistance;
    }
    if ((0 - translate) > (lineLength - containerWidth)) translate = containerWidth - lineLength;
    if (translate > 0) translate = 0;

    line.style.transform = 'translateX(' + translate + 'px)';
    // update the navigation items status (toggle inactive class)
    (translate == 0) ? Util.addClass(navigation[0], 'cd-h-timeline__navigation--inactive'):
        Util.removeClass(navigation[0], 'cd-h-timeline__navigation--inactive');
    (translate == containerWidth - lineLength) ? Util.addClass(navigation[1], 'cd-h-timeline__navigation--inactive'): Util.removeClass(navigation[1], 'cd-h-timeline__navigation--inactive');
};

function selectNewDate(newDateIndex: number, date: NodeListOf < HTMLElement > , oldDateIndex: number, target: any) { // ned date has been selected -> update timeline
    newDateIndex = Util.getIndexInArray(date, target);
    oldDateIndex = Util.getIndexInArray(date, $selectedDate);
    Util.removeClass($selectedDate, 'cd-h-timeline__date--selected');
    Util.addClass(date[newDateIndex], 'cd-h-timeline__date--selected');
    $selectedDate = date[newDateIndex];
    updateOlderEvents(date, newDateIndex);
    updateVisibleContent(newDateIndex, oldDateIndex);
    updateFilling();
};

let element: E,
    datesContainer: E, content: NodeListOf < E > , navigation: NodeListOf < E > , newDateIndex: number, oldDateIndex: number;
let line: E; // grey line in the top timeline section
let fillingLine: E; // green filling line in the top timeline section  
let contentWrapper: E;

let eventsMinDistance = 60; // min distance between two consecutive events (in px)
let eventsMaxDistance = 200; // max distance between two consecutive events (in px)
let translate = 0; // this will be used to store the translate value of {line}
let lineLength = 0; //total length of {line}

const selectedDate = writable < E | null > (null);

onMount(() => {
    content = contentWrapper.querySelectorAll < HTMLElement > ('.cd-h-timeline__event') !;
    navigation = element.querySelectorAll < HTMLElement > ('.cd-h-timeline__navigation') !;
    $selectedDate = line.querySelector < HTMLElement > ('.cd-h-timeline__date--selected') !;
    let date = line.querySelectorAll < HTMLElement > ('.cd-h-timeline__date') !;

    // store index of selected and previous selected dates
    oldDateIndex = Util.getIndexInArray(date, selectedDate);
    newDateIndex = oldDateIndex;

    let dateValues = parseDate(date);
    let minLapse = calcMinLapse(dateValues);

    initTimeline(dateValues, date, minLapse);
    initEvents(navigation, date, content, newDateIndex, oldDateIndex);
    console.log(line)
    // navigate the timeline when inside the viewport using the keyboard
    document.addEventListener('keydown', function(event) {
        if ((event.keyCode && event.keyCode == 39) || (event.key && event.key.toLowerCase() == 'arrowright')) {
            if (elementInViewport(element)) keyNavigateTimeline(date, oldDateIndex, newDateIndex, 'next'); // move to next event
        } else if ((event.keyCode && event.keyCode == 37) || (event.key && event.key.toLowerCase() == 'arrowleft')) {
            if (elementInViewport(element)) keyNavigateTimeline(date, oldDateIndex, newDateIndex, 'prev'); // move to prev event
        }
    });
});

/*

How to tell if a DOM element is visible in the current viewport?

http://stackoverflow.com/questions/123999/how-to-tell-if-a-dom-element-is-visible-in-the-current-viewport

*/
function elementInViewport(el: E) {
    var top = el.offsetTop;
    var left = el.offsetLeft;
    var width = el.offsetWidth;
    var height = el.offsetHeight;

    while (el.offsetParent) {
        el = el.offsetParent as E;
        top += el.offsetTop;
        left += el.offsetLeft;
    }

    return (
        top < (window.pageYOffset + window.innerHeight) &&
        left < (window.pageXOffset + window.innerWidth) &&
        (top + height) > window.pageYOffset &&
        (left + width) > window.pageXOffset
    );
}

function updateOlderEvents(date: string | any[] | NodeListOf < HTMLElement > , newDateIndex: number) { // update older events style
    for (var i = 0; i < date.length; i++) {
        (i < newDateIndex) ? Util.addClass(date[i], 'cd-h-timeline__date--older-event'):
            Util.removeClass(date[i], 'cd-h-timeline__date--older-event');
    }
};

function updateVisibleContent(newDateIndex: number, oldDateIndex: number) { // show content of new selected date
    let classEntering = 'cd-h-timeline__event--selected',
        classLeaving = '';
    if (newDateIndex > oldDateIndex) {
        classEntering = 'cd-h-timeline__event--selected cd-h-timeline__event--enter-right',
            classLeaving = 'cd-h-timeline__event--leave-left';
    } else if (newDateIndex < oldDateIndex) {
        classEntering = 'cd-h-timeline__event--selected cd-h-timeline__event--enter-left',
            classLeaving = 'cd-h-timeline__event--leave-right';
    }

    Util.addClass(content[newDateIndex], classEntering);
    if (newDateIndex != oldDateIndex) {
        Util.removeClass(content[oldDateIndex], 'cd-h-timeline__event--selected');
        Util.addClass(content[oldDateIndex], classLeaving);
        contentWrapper.style.height = content[newDateIndex].offsetHeight + 'px';
    }
};

function resetAnimation(newDateIndex: number, oldDateIndex: number) { // reset content classes when entering animation is over
    contentWrapper.style.height = "";
    Util.removeClass(content[newDateIndex], 'cd-h-timeline__event--enter-right cd-h-timeline__event--enter-left');
    Util.removeClass(content[oldDateIndex], 'cd-h-timeline__event--leave-right cd-h-timeline__event--leave-left');
};

function keyNavigateTimeline(date: NodeListOf < HTMLElement > , oldDateIndex: number, newDateIndex: number, direction: string) { // navigate the timeline using the keyboard
    var newIndex = (direction == 'next') ? newDateIndex + 1 : newDateIndex - 1;
    if (newIndex < 0 || newIndex >= date.length) return;
    selectNewDate(newIndex, date, oldDateIndex, date[newIndex]);
    resetTimelinePosition(direction);
};

function resetTimelinePosition(direction: string) { //translate timeline according to new selected event position
    var eventStyle = window.getComputedStyle($selectedDate!, null),
        eventLeft = Number(eventStyle.getPropertyValue('left').replace('px', '')),
        timelineWidth = datesContainer.offsetWidth;

    if ((direction == 'next' && eventLeft >= timelineWidth - translate) || (direction == 'prev' && eventLeft <= -translate)) {
        translate = timelineWidth / 2 - eventLeft;
        translateTimeline(navigation, "");
    }
};

function parseDate(date: NodeListOf < E > ) { // get timestamp value for each date
    let dateArrays = [],
        dayComp, timeComp;
    for (let i = 0; i < date.length; i++) {
        let singleDate = date[i].getAttribute('data-date'),
            dateComp = singleDate!.split('T');

        if (dateComp.length > 1) { //both DD/MM/YEAR and time are provided
            dayComp = dateComp[0].split('/'),
                timeComp = dateComp[1].split(':');
        } else if (dateComp[0].indexOf(':') >= 0) { //only time is provide
            dayComp = ["2000", "0", "0"],
                timeComp = dateComp[0].split(':');
        } else { //only DD/MM/YEAR
            dayComp = dateComp[0].split('/'),
                timeComp = ["0", "0"];
        }
        let newDate = new Date(parseInt(dayComp[2]), parseInt(dayComp[1]) - 1, parseInt(dayComp[0]), parseInt(timeComp[0]), parseInt(timeComp[1]));
        dateArrays.push(newDate);
    }
    return dateArrays;
};

function calcMinLapse(dateValues: Date[]) { // determine the minimum distance among events
    let dateDistances = [];
    for (let i = 1; i < dateValues.length; i++) {
        let distance = daydiff(dateValues[i - 1], dateValues[i]);
        if (distance > 0) dateDistances.push(distance);
    }

    return (dateDistances.length > 0) ? Math.min.apply(null, dateDistances) : 86400000;
};

function daydiff(first: Date, second: Date) { // time distance between events
    console.log(first +" & "+ second.getTime());
    
    return Math.round(second.getTime() - first.getTime());
};
</script>

<section bind:this={element} class="cd-h-timeline js-cd-h-timeline margin-bottom-md">

    <div bind:this={datesContainer} class="cd-h-timeline__container container">
        <div class="cd-h-timeline__dates">
            <div bind:this={line} class="cd-h-timeline__line">
                <ol>
                    <li><a href="#0" data-date="16/01/2014" class="cd-h-timeline__date cd-h-timeline__date--selected">16 Jan</a></li>
                    <li><a href="#0" data-date="28/02/2014" class="cd-h-timeline__date">28 Feb</a></li>
                    <li><a href="#0" data-date="20/04/2014" class="cd-h-timeline__date">20 Mar</a></li>
                    <li><a href="#0" data-date="20/05/2014" class="cd-h-timeline__date">20 May</a></li>
                    <li><a href="#0" data-date="09/07/2014" class="cd-h-timeline__date">09 Jul</a></li>
                    <li><a href="#0" data-date="30/08/2014" class="cd-h-timeline__date">30 Aug</a></li>
                    <li><a href="#0" data-date="15/09/2014" class="cd-h-timeline__date">15 Sep</a></li>
                    <li><a href="#0" data-date="01/11/2014" class="cd-h-timeline__date">01 Nov</a></li>
                    <li><a href="#0" data-date="10/12/2014" class="cd-h-timeline__date">10 Dec</a></li>
                    <li><a href="#0" data-date="19/01/2015" class="cd-h-timeline__date">29 Jan</a></li>
                    <li><a href="#0" data-date="03/03/2015" class="cd-h-timeline__date">3 Mar</a></li>
                </ol>

                <span bind:this={fillingLine} class="cd-h-timeline__filling-line" aria-hidden="true"></span>
            </div> <!-- .cd-h-timeline__line -->
        </div> <!-- .cd-h-timeline__dates -->

        <ul>
            <li><a href="#0" class="text-replace cd-h-timeline__navigation cd-h-timeline__navigation--prev cd-h-timeline__navigation--inactive">Prev</a></li>
            <li><a href="#0" class="text-replace cd-h-timeline__navigation cd-h-timeline__navigation--next">Next</a></li>
        </ul>
    </div> <!-- .cd-h-timeline__container -->
    <div bind:this={contentWrapper} class="cd-h-timeline__events">
        <ol>
            <li class="cd-h-timeline__event cd-h-timeline__event--selected text-component">
                <div class="cd-h-timeline__event-content container">
                    <h1 class="cd-h-timeline__event-title">Horizontal Timeline</h1>
                    <em class="cd-h-timeline__event-date">January 16th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">February 28th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">March 20th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">May 20th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">July 9th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">August 30th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">September 15th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">November 1st, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">December 10th, 2014</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">January 29th, 2015</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="cd-h-timeline__event text-component">
                <div class="cd-h-timeline__event-content container">
                    <h2 class="cd-h-timeline__event-title">Event title here</h2>
                    <em class="cd-h-timeline__event-date">March 3rd, 2015</em>
                    <p class="cd-h-timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>
        </ol>
    </div> <!-- .cd-h-timeline__events -->
</section>

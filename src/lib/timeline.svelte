<script context="module" lang="ts">
    export const dates = writable < Array < number >> ([]); 
    export function parseDate(date: string) { // get timestamp value for each date
        let dayComp, timeComp,
            dateComp = date.split('T');

        if (dateComp.length > 1) { //both DD/MM/YEAR and time are provided
            dayComp = dateComp[0].split('/')
            timeComp = dateComp[1].split(':');
        } else if (dateComp[0].indexOf(':') >= 0) { //only time is provide
            dayComp = ["2000", "0", "0"]
            timeComp = dateComp[0].split(':');
        } else { //only DD/MM/YEAR
            dayComp = dateComp[0].split('/')
            timeComp = ["0", "0"];
        }
        return new Date(parseInt(dayComp[2]), parseInt(dayComp[1]) - 1, parseInt(dayComp[0]), parseInt(timeComp[0]), parseInt(timeComp[1]));
    };
</script> 
<script lang="ts" >
    import {onMount } from "svelte";
    import type { E }  from "./type.js";
import {
    Util
} from "./util.js";
import {
    writable
} from "svelte/store";
import {
    SwipeContent
} from "./swipe.js";
	import Event from "./event.svelte";

let element: E,
    datesContainer: E, content: NodeListOf < E > , navigation: NodeListOf < E > , newDateIndex: number, 
    oldDateIndex: number,containerWidth:number;
let line: E; // grey line in the top timeline section
let fillingLine: E; // green filling line in the top timeline section  
let contentWrapper: E;

let eventsMinDistance = 60; // min distance between two consecutive events (in px)
let eventsMaxDistance = 200; // max distance between two consecutive events (in px)
let translate = 0; // this will be used to store the translate value of {line}
let lineLength = 0; //total length of {line}

const selectedDate = writable < number > (0);

onMount(() => {
    content = contentWrapper.querySelectorAll < HTMLElement > ('.timeline__event') !;
    navigation = element.querySelectorAll < HTMLElement > ('.timeline__navigation') !;
    $selectedDate = $dates[0];
    // let date = line.querySelectorAll < HTMLElement > ('.timeline__date') !;

    // store index of selected and previous selected dates
    oldDateIndex = Util.getIndexInArray($dates, $selectedDate);
    newDateIndex = oldDateIndex;

    lineLength = ($dates.length * eventsMaxDistance)  + eventsMinDistance;
    line.style.width = (lineLength) + 'px';
    selectNewDate(newDateIndex);
    resetTimelinePosition('next');
    initEvents(content, newDateIndex, oldDateIndex);
    Util.addClass(element, 'timeline--loaded');
    // navigate the timeline when inside the viewport using the keyboard
    document.addEventListener('keydown', function(event) {
        if ((event.keyCode && event.keyCode == 39) || (event.key && event.key.toLowerCase() == 'arrowright')) {
            if (elementInViewport(element)) keyNavigateTimeline('next'); // move to next event
        } else if ((event.keyCode && event.keyCode == 37) || (event.key && event.key.toLowerCase() == 'arrowleft')) {
            if (elementInViewport(element)) keyNavigateTimeline('prev'); // move to prev event
        }
    });
});

function formatDate(date:number|string) {
    return new Date(date).toDateString()
}

function initEvents(content: NodeListOf < E > , newDateIndex: number, oldDateIndex: any) {
    //swipe on timeline
    new SwipeContent(datesContainer);
    datesContainer.addEventListener('swipeLeft', function(event: any) {
        translateTimeline('next');
    });
    datesContainer.addEventListener('swipeRight', function(event: any) {
        translateTimeline('prev');
    });
};

function updateFilling() { // update fillingLine scale value
    let left = $dates.indexOf($selectedDate) +1;
    left = (left * eventsMaxDistance) + 40;
    fillingLine.style.transform = 'scaleX(' + (left / lineLength) + ')';
};

function translateTimeline(direction: string) { // translate timeline (and date elements)
    if (direction) {
        translate = (direction == 'next') ? translate - containerWidth + eventsMinDistance : translate + containerWidth - eventsMinDistance;
    }
    if ((0 - translate) > (lineLength - containerWidth)) translate = containerWidth - lineLength;
    if (translate > 0) translate = 0;

    line.style.transform = 'translateX(' + translate + 'px)';
};

function selectNewDate(target: number) {
    oldDateIndex = newDateIndex; // ned date has been selected -> update timeline
    newDateIndex = target;
    $selectedDate = $dates[newDateIndex];
    updateFilling();
};

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

function keyNavigateTimeline(direction: string) { // navigate the timeline using the keyboard
    var newIndex = (direction == 'next') ? newDateIndex + 1 : newDateIndex - 1;
    if (newIndex < 0 || newIndex >= $dates.length) return;
    selectNewDate(newIndex);
    resetTimelinePosition(direction);
};

function resetTimelinePosition(direction: string) { //translate timeline according to new selected event position
    let eventLeft = $dates.indexOf($selectedDate) * 200;
    if ((direction == 'next' && eventLeft >= containerWidth - translate) || (direction == 'prev' && eventLeft <= -translate)) {
        translate = containerWidth / 2 - eventLeft;
        translateTimeline("");
    }
};

function calcMinLapse(dates: number[]) { // determine the minimum distance among events
    let dateDistances = [];
    for (let i = 1; i < dates.length; i++) {
        let distance = daydiff(dates[i - 1], dates[i]);
        if (distance > 0) dateDistances.push(distance);
    }

    return (dateDistances.length > 0) ? Math.min.apply(null, dateDistances) : 86400000;
};

function daydiff(first: number, second: number) { // time distance between events
    return Math.round(second- first);
};
</script>

<section bind:this={element} class="timeline js-timeline margin-bottom-md">

    <div class="timeline__container container">
        <div bind:this={datesContainer} class="timeline__dates" bind:offsetWidth={containerWidth}>
            <div bind:this={line} class="timeline__line">
                <ol>
                    {#if $dates}
                        {#each $dates as item, index}
                            <li><a on:click={() => {selectNewDate(index);}} href="#0" style="left : {(index + 1)*eventsMaxDistance}px" class:timeline__date--selected={$selectedDate === item} 
                            class="timeline__date" class:timeline__date--older-event={index < newDateIndex}>{formatDate(item)}</a></li>
                        {/each}
                    {/if}
                </ol>

                <span bind:this={fillingLine} class="timeline__filling-line" aria-hidden="true"></span>
            </div> <!-- .timeline__line -->
        </div> <!-- .timeline__dates -->

        <ul>
            <li><a on:click={() => {translateTimeline('prev');}} href="#0" class:timeline__navigation--inactive={translate === 0} class="text-replace timeline__navigation timeline__navigation--prev">Prev</a></li>
            <li><a on:click={() => {translateTimeline('next');}} class:timeline__navigation--inactive={translate == containerWidth - lineLength} href="#0" class="text-replace timeline__navigation timeline__navigation--next">Next</a></li>
        </ul>
    </div> <!-- .timeline__container -->
    <div bind:this={contentWrapper} class="timeline__events">
        <ol>
            {#each [0,1,2,3,4,5,6,7,8,9] as item}
                 <Event {oldDateIndex} {newDateIndex} date={`28/${item+1}/2014`} />
            {/each}
        </ol>
    </div> <!-- .timeline__events -->
</section>

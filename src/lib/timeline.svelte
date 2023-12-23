<script context="module" lang="ts">
    export const dates = writable < Array < number >> ([]);
    export const newDateIndex = writable<number>(0)
    export const oldDateIndex = writable<number>(0)
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
    import {onMount} from "svelte";
    import type { E }  from "./type.js";
    import "./style.css";
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
    datesContainer: E, content: NodeListOf < E >,containerWidth:number;
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
    $selectedDate = $dates[0];

    // store index of selected and previous selected dates
    $oldDateIndex = Util.getIndexInArray($dates, $selectedDate);
    $newDateIndex = $oldDateIndex;

    lineLength = ($dates.length * eventsMaxDistance)  + eventsMinDistance;
    line.style.width = (lineLength) + 'px';
    selectNewDate($newDateIndex);
    resetTimelinePosition('next');
    initEvents();
    Util.addClass(element, 'loaded');
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

function initEvents() {
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
    let left = $newDateIndex +1;
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
    $oldDateIndex = $newDateIndex; // ned date has been selected -> update timeline
    $newDateIndex = target;
    $selectedDate = $dates[$newDateIndex];
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
    var newIndex = (direction == 'next') ? $newDateIndex + 1 : $newDateIndex - 1;
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

function daydiff(first: number, second: number) { // time distance between events
    return Math.round(second- first);
};
</script>

<section bind:this={element} class="svelte-timeline style-one">
    
    <div bind:this={contentWrapper} class="events-content">
        <ol style="list-style-type: none;">
            {#each [0,1,2,3,4,5,6,7,8,9] as item}
                <Event date={`28/${item+1}/2014`} />
            {/each}
        </ol>
    </div> <!-- .timeline__events -->
    <div class="timeline">
        <div bind:this={datesContainer} class="events-wrapper" bind:offsetWidth={containerWidth}>
            <div bind:this={line} class="events">
                <ol style="list-style-type: none;">
                    {#if $dates}
                        {#each $dates as item, index}
                            <li><a on:click={() => {selectNewDate(index);}} href="#0" style="left : {(index + 1)*eventsMaxDistance}px" class:selected={$selectedDate === item} class:older-event={index < $newDateIndex}>{formatDate(item)}</a></li>
                        {/each}
                    {/if}
                </ol>

                <span bind:this={fillingLine} class="filling-line" aria-hidden="true"></span>
            </div> <!-- .timeline__line -->
        </div> <!-- .timeline__dates -->

        <ul class="svelte-timeline-navigation" style="list-style-type: none;">
            <li><a on:click={() => {translateTimeline('prev');}} href="#0" class:inactive={translate === 0} class="text-replace prev">Prev</a></li>
            <li><a on:click={() => {translateTimeline('next');}} class:inactive={translate == containerWidth - lineLength} href="#0" class="text-replace next">Next</a></li>
        </ul> <!-- navigations -->
    </div> <!-- .timeline__container -->
</section>

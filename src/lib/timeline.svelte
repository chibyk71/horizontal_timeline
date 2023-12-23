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

    // let dateValues = parseDate(date);
    let minLapse = calcMinLapse($dates);
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

// function initTimeline(dates: number[], date: NodeListOf < E > , minLapse: number) {
//     // set dates left position
//     // let left = 0;
//     // for (let i = 0; i < dates.length; i++) {
//     //     let j = (i == 0) ? 0 : i - 1;
//     //     let distance = daydiff(dates[j], dates[i]),
//     //         distanceNorm = (Math.round(distance / minLapse) + 2) * eventsMinDistance;
//     //     // console.log(distance+" "+ minLapse);
//     //     if (distanceNorm < eventsMinDistance) {
//     //         distanceNorm = eventsMinDistance;
//     //     } else if (distanceNorm > eventsMaxDistance) {
//     //         distanceNorm = eventsMaxDistance;
//     //     }
//     //     left += distanceNorm;
//     //     date[i].setAttribute('style', 'left:' + left + 'px');

//     // }

//     // set line/filling line dimensions
//     // todo: line.style.width = (left + eventsMinDistance) + 'px';
//     // console.log(line);
//     // reveal timeline
//     // Util.addClass(element, 'timeline--loaded');
// };

function initEvents(content: NodeListOf < E > , newDateIndex: number, oldDateIndex: any) {
    //swipe on timeline
    new SwipeContent(datesContainer);
    datesContainer.addEventListener('swipeLeft', function(event: any) {
        translateTimeline('next');
    });
    datesContainer.addEventListener('swipeRight', function(event: any) {
        translateTimeline('prev');
    });

    //select a new event
    for (let i = 0; i < content.length; i++) {
        content[i].addEventListener('animationend', function(event) {
            if (i == newDateIndex && newDateIndex != oldDateIndex) resetAnimation(newDateIndex, oldDateIndex);
        });
    }
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
    // updateOlderEvents($dates, newDateIndex);
    // updateVisibleContent(newDateIndex, oldDateIndex);
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

// function updateOlderEvents(date: string | any[] | NodeListOf < HTMLElement > , newDateIndex: number) { // update older events style
//     for (var i = 0; i < date.length; i++) {
//         (i < newDateIndex) ? Util.addClass(date[i], 'timeline__date--older-event'):
//             Util.removeClass(date[i], 'timeline__date--older-event');
//     }
// };

// function updateVisibleContent(newDateIndex: number, oldDateIndex: number) { // show content of new selected date
//     let classEntering = 'selected',
//         classLeaving = '';
//     if (newDateIndex > oldDateIndex) {
//         classEntering = 'selected enter-right',
//             classLeaving = 'leave-left';
//     } else if (newDateIndex < oldDateIndex) {
//         classEntering = 'selected enter-left',
//             classLeaving = 'leave-right';
//     }

//     Util.addClass(content[newDateIndex], classEntering);
//     if (newDateIndex != oldDateIndex) {
//         Util.removeClass(content[oldDateIndex], 'selected');
//         Util.addClass(content[oldDateIndex], classLeaving);
//         contentWrapper.style.height = content[newDateIndex].offsetHeight + 'px';
//     }
// };

function resetAnimation(newDateIndex: number, oldDateIndex: number) { // reset content classes when entering animation is over
    contentWrapper.style.height = "";
    Util.removeClass(content[newDateIndex], 'timeline__event--enter-right timeline__event--enter-left');
    Util.removeClass(content[oldDateIndex], 'timeline__event--leave-right timeline__event--leave-left');
};

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
                    <!-- <li><a href="#0" data-date="16/01/2014" class="timeline__date timeline__date--selected">16 Jan</a></li>
                    <li><a href="#0" data-date="28/02/2014" class="timeline__date">28 Feb</a></li>
                    <li><a h/ref="#0" data-date="20/04/2014" class="timeline__date">20 Mar</a></li>
                    <li><a href="#0" data-date="20/05/2014" class="timeline__date">20 May</a></li>
                    <li><a href="#0" data-date="09/07/2014" class="timeline__date">09 Jul</a></li>
                    <li><a href="#0" data-date="30/08/2014" class="timeline__date">30 Aug</a></li>
                    <li><a href="#0" data-date="15/09/2014" class="timeline__date">15 Sep</a></li>
                    <li><a href="#0" data-date="01/11/2014" class="timeline__date">01 Nov</a></li>
                    <li><a href="#0" data-date="10/12/2014" class="timeline__date">10 Dec</a></li>
                    <li><a href="#0" data-date="19/01/2015" class="timeline__date">29 Jan</a></li>
                    <li><a href="#0" data-date="03/03/2015" class="timeline__date">3 Mar</a></li> -->
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
            <!-- <li class="timeline__event timeline__event--selected text-component">
                <div class="timeline__event-content container">
                    <h1 class="timeline__event-title">Horizontal Timeline</h1>
                    <em class="timeline__event-date">January 16th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">February 28th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">March 20th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">May 20th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">July 9th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">August 30th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">September 15th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">November 1st, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">December 10th, 2014</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">January 29th, 2015</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li>

            <li class="timeline__event text-component">
                <div class="timeline__event-content container">
                    <h2 class="timeline__event-title">Event title here</h2>
                    <em class="timeline__event-date">March 3rd, 2015</em>
                    <p class="timeline__event-description color-contrast-medium">
                        Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
                    </p>
                </div>
            </li> -->
        </ol>
    </div> <!-- .timeline__events -->
</section>

<script lang="ts">
	import { onMount } from "svelte";
    import {  dates, parseDate } from "$lib/timeline.svelte"
	import { writable } from "svelte/store";
	
    export let date: number|string = "";
    export let newDateIndex:number;
    export let oldDateIndex:number;
	let id:number,classEntering:string,classLeaving:string,animationEnded = writable(false);
	$:{if (newDateIndex > oldDateIndex) {
		classEntering = 'timeline__event--selected timeline__event--enter-right'
		classLeaving = 'timeline__event--leave-left';
	} else if(newDateIndex < oldDateIndex) {
		classEntering = 'timeline__event--selected timeline__event--enter-left'
		classLeaving = 'timeline__event--leave-right';
	} else {
		classEntering = 'timeline__event--selected'
		classLeaving = '';
	}}
	// function resetAnimation() { // reset content classes when entering animation is over
	// 	contentWrapper.style.height = "";
	// 	Util.removeClass(content[newDateIndex], 'timeline__event--enter-right timeline__event--enter-left');
	// 	Util.removeClass(content[oldDateIndex], 'timeline__event--leave-right timeline__event--leave-left');
	// {if (id == newDateIndex && newDateIndex != oldDateIndex) resetAnimation(newDateIndex, oldDateIndex);}};
    onMount(() =>{
		date = new Date(parseDate(date as string)).getTime();
        $dates.push(date)
		id = $dates.indexOf(date);
    })
</script>
<div class="timeline__event text-component {id == newDateIndex ? classEntering: classLeaving}" on:animationstart={() =>  $animationEnded = false} on:animationend={() => $animationEnded = true }  class:timeline__event--leave-right={!$animationEnded && (id == oldDateIndex)} class:timeline__event--leave-left={!$animationEnded && (id == oldDateIndex)}>
    <div class="timeline__event-content container">
        <h1 class="timeline__event-title">Horizontal Timeline</h1>
        <em class="timeline__event-date">{new Date(date).toDateString()}</em>
        <p class="timeline__event-description color-contrast-medium">
            Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
        </p>
    </div>
</div>
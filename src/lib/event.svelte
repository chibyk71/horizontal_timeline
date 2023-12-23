<script lang="ts">
	import { onMount } from "svelte";
    import {  dates, newDateIndex, oldDateIndex, parseDate } from "$lib/timeline.svelte"
	import { writable } from "svelte/store";
	
    export let date: number|string = "";
	let id:number,classEntering:string,classLeaving:string,animationEnded = writable(false);
	$:{if ($newDateIndex > $oldDateIndex) {
		classEntering = 'selected enter-right'
		classLeaving = 'leave-left';
	} else if($newDateIndex < $oldDateIndex) {
		classEntering = 'selected enter-left'
		classLeaving = 'leave-right';
	} else {
		classEntering = 'selected'
		classLeaving = '';
	}}
	
    onMount(() =>{
		date = new Date(parseDate(date as string)).getTime();
        $dates.push(date)
		id = $dates.indexOf(date);
    })
</script>
<li class="{id == $newDateIndex ? classEntering: classLeaving}" on:animationstart={() =>  $animationEnded = false} on:animationend={() => $animationEnded = true }  class:leave-right={!$animationEnded && (id == $oldDateIndex)} class:leave-left={!$animationEnded && (id == $oldDateIndex)}>
	<div class="timeline__event text-component">
		<slot name="event">
			<div class="timeline__event-content container">
				<h1 class="timeline__event-title">Horizontal Timeline</h1>
				<em class="timeline__event-date">{new Date(date).toDateString()}</em>
				<p class="timeline__event-description color-contrast-medium">
					Lorem ipsum dolor sit amet, consectetur adipisicing elit. Illum praesentium officia, fugit recusandae ipsa, quia velit nulla adipisci? Consequuntur aspernatur at, eaque hic repellendus sit dicta consequatur quae, ut harum ipsam molestias maxime non nisi reiciendis eligendi! Doloremque quia pariatur harum ea amet quibusdam quisquam, quae, temporibus dolores porro doloribus.
				</p>
			</div>
		</slot>
	</div>
</li>
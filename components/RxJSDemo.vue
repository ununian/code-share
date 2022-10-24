<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { useObservable, from } from '@vueuse/rxjs';
import { templateRef } from '@vueuse/core';
import { combineLatest, fromEvent, of, scan, startWith, switchMap } from 'rxjs';

const add = templateRef<HTMLButtonElement>('add');
const reset = templateRef<HTMLButtonElement>('reset');
const label = templateRef<HTMLSpanElement>('label');

onMounted(() => {
  fromEvent(reset.value!, 'click')
    .pipe(
      startWith(0),
      switchMap(() =>
        fromEvent(add.value!, 'click').pipe(
          scan((acc) => acc + 1, 0),
          startWith(0),
        )
      )
    )
    .subscribe((v) => {
      label.value!.innerText = v.toString();
    });
});
</script>

<template>
  <div class="rx-js">
    <button ref="add">Add</button>
    <button ref="reset">Reset</button>
    <span ref="label"></span>
  </div>
</template>
<style>
.rx-js * {
  margin-right: 1em;
  border: 1px solid #ccc;
}
</style>

<template>
  <div class="game-screen">
    <div class="game-screen-section">
      <button class="quit" @click="quit">
        {{$t('ABORT')}}
      </button>
      <ScoreLine 
        :result="result" 
        :timeLeft="timeLeft"
        :isInfiniteRound="!options.gameLength" />
      <FeedbackLine 
        :feedback="feedback"
        :feedbackNote="feedbackNote"
        :uniqueId="numAnswers" />
    </div>
    <div class="game-screen-section" id="game-note-display">
      <NoteDisplay 
        :currentExercise="currentExercise"
        @play="playNote(currentExercise.value)" />
    </div>
    <div class="game-screen-section" id="game-screen-input">
      <ButtonInput 
        v-if="options.inputMode === 'button'" 
        :isSharp="currentExercise.isSharp" 
        @solved="checkAnswer" />  
      <KeyboardInput 
        v-if="options.inputMode === 'keyboard'" 
        @solved="checkAnswer" />
      <MidiInput
        v-if="options.inputMode === 'midi'"
        @solved="(value) => checkAnswer(value, true)" />
      <MousetrapInput 
        v-if="options.inputMode !== 'midi'"
        @solved="checkAnswer" />
    </div>
  </div>
</template>

<script>
import ScoreLine from './ScoreLine';
import NoteDisplay from './NoteDisplay';
import FeedbackLine from './FeedbackLine';
import ButtonInput from './ButtonInput';
import KeyboardInput from './KeyboardInput';
import MidiInput from './MidiInput';
import MousetrapInput from './MousetrapInput';

import Utils from '../model/Utils';
import Options from '../model/Options';
import Statistics from '../model/Statistics';

import * as _ from 'lodash';

export default {
  name: 'Game',
  components: { 
    ScoreLine,
    NoteDisplay,
    FeedbackLine,
    ButtonInput,
    KeyboardInput,
    MidiInput,
    MousetrapInput,
  },
  data () {
    return {
      options: Options,
      currentExercise: {
        clef: 'treble',
        staff: 'treble',
        value: 36,
        isSharp: false
      },
      numCorrect: 0,
      numWrong: 0,
      timeLeft: 0,
      timer: null,
      feedbackNote: 'none',
      feedback: 'none',
      sample: null,
    }
  },
  computed: {
    numAnswers(){
      return this.numCorrect + this.numWrong;
    },
    accuracy(){
      if(this.numAnswers === 0){
        return 0;
      }
      return Math.round(100 * this.numCorrect / this.numAnswers);
    },
    score(){
      if(this.numAnswers === 0){
        return 0;
      }
      return Math.round(this.baseFactor * this.numCorrect * this.numCorrect / this.numAnswers);
    },
    baseFactor(){
      // 20s -> 300, 1min -> 100, 5min -> 20
      return this.options.gameLength ? (6000 / this.options.gameLength) : 0;
    },
    result(){
      return {
        numAnswers: this.numAnswers,
        numCorrect: this.numCorrect,
        numWrong: this.numWrong,
        accuracy: this.accuracy,
        score: this.score
      }
    },
    minBassValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 36;
        case 'c7c8':
          return 36;
        case 'easy':
          return 36;
        case 'normal':
          return 36;
        case 'hard':
          return 24;
        case 'hard2':
        default:
          return 9;
      }
    },
    maxBassValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 48;
        case 'c7c8':
          return 48;
        case 'easy':
          return 48;
        case 'normal':
          return 48;
        case 'hard':
          return 57;
        case 'hard2':
        default:
          return 57;
      }
    },
    minTrebleValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 72;
        case 'c7c8':
          return 84;
        case 'easy':
          return 48;
        case 'normal':
          return 48;
        case 'hard':
          return 40;
        case 'hard2':
        default:
          return 40;
      }
    },
    maxTrebleValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 84;
        case 'c7c8':
          return 96;
        case 'easy':
          return 60;
        case 'normal':
          return 69;
        case 'hard':
          return 84;
        case 'hard2':
        default:
          return 96;
      }
    },
    minAltoValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 43;
        case 'c7c8':
          return 43;
        case 'easy':
          return 43;
        case 'normal':
          return 36;
        case 'hard':
          return 29;
        case 'hard2':
        default:
          return 29;
      }
    },
    maxAltoValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 55;
        case 'c7c8':
          return 55;
        case 'easy':
          return 55;
        case 'normal':
          return 60;
        case 'hard':
          return 67;
        case 'hard2':
        default:
          return 67;
      }
    },
    minTenorValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 43;
        case 'c7c8':
          return 43;
        case 'easy':
          return 43;
        case 'normal':
          return 36;
        case 'hard':
          return 26;
        case 'hard2':
        default:
          return 26;
      }
    },
    maxTenorValue(){
      switch(this.options.difficulty){
        case 'c6c7':
          return 55;
        case 'c7c8':
          return 55;
        case 'easy':
          return 55;
        case 'normal':
          return 60;
        case 'hard':
          return 64;
        case 'hard2':
        default:
          return 64;
      }
    }
  },
  methods: {
    startGame(){
      this.numCorrect = 0;
      this.numWrong = 0;
      this.timeLeft = this.options.gameLength;
      if (this.options.gameLength){
        this.timer = setInterval(() => {
            this.timeLeft -= 1;
            if(this.timeLeft < 0){
              this.onGameFinished();
            }
          }, 1000);
      }
      this.generateNewExercise();
    },
    onExit(){
      clearInterval(this.timer);
      if(this.sample){
        this.sample.pause();
      }
    },
    onGameFinished(){
      this.onExit();
      Statistics.addScore(this.score,
                          this.options.clef,
                          this.options.difficulty,
                          this.options.accidentals);
      this.$emit('gameEnded', this.result); 
    },
    quit(){
      this.onExit();
      this.$emit('gameEnded', null);
    },
    generateNewExercise(){
      let exercise = this.currentExercise;
      while(exercise.value === this.currentExercise.value){
        exercise = this.generateExercise();
      }
      this.currentExercise = exercise;
    },
    generateExercise(){
      var clef = _.sample(this.options.clef);
      if(clef === 'piano'){
        const staves = ['treble', 'bass'];
        clef = staves[_.random(0, staves.length - 1)];
        var staff = 'piano';
      }else{
        var staff = clef;
      }
      const exercise = { clef, staff, value: this.getRandomNoteForClef(clef) };
      switch(this.options.accidentals){
        case 'no':
          exercise.isSharp = true;
          if(Utils.hasAccidental(exercise.value)){
            _.sample([true, false]) ? exercise.value++ : exercise.value--;
          }
          break;
        case 'onlySharp':
          exercise.isSharp = true;
          break;
        case 'onlyFlat':
          exercise.isSharp = false;
          break;
        case 'sharpAndFlat':
          exercise.isSharp = _.sample([true, false])
          break;
      }
      return exercise;
    },
    getRandomNoteForClef(clef){
      switch(clef){
        case 'treble': 
          return _.random(this.minTrebleValue, this.maxTrebleValue);
        case 'bass':
          return _.random(this.minBassValue, this.maxBassValue);
        case 'alto':
          return _.random(this.minAltoValue, this.maxAltoValue);
        case 'tenor':
          return _.random(this.minTenorValue, this.maxTenorValue);
      }
    },
    checkAnswer(value, checkOctave = false){
      this.playNote(Utils.getNearestNoteOfValue(value, this.currentExercise.value));
      if(checkOctave){
        var submittedValue = value;
        var expectedValue = this.currentExercise.value;
      } else {
        var submittedValue = value % 12;
        var expectedValue = this.currentExercise.value % 12;
      }
      if(submittedValue === expectedValue){
        this.onCorrectAnswer(value, this.currentExercise.isSharp);
        this.generateNewExercise();
      } else {
        this.onWrongAnswer(value);
      }
    },
    onCorrectAnswer(noteValue, isSharp){
      this.numCorrect += 1;

      if(this.options.displayNote) {
        this.feedback = 'correct-note';
        this.feedbackNote = Utils.getNoteName(noteValue % 12, isSharp);
      }else{
        this.feedback = 'correct';
      }
    },
    onWrongAnswer(wrongValue){
      this.numWrong += 1;
      this.feedback = 'wrong';
      if('vibrate' in navigator && this.options.vibration) {
        navigator.vibrate(200);
      }
    },
    playNote(value){
      if(!this.options.sound){
        return;
      }
      if(!!this.sample && !this.sample.paused){
        this.sample.pause();
      }
      this.sample = new Audio('static/samples/piano/' + value + '.mp3');
      this.sample.play();
    }
  },
  mounted(){
    this.startGame();
  }
}
</script>

<style>
.game-screen {
  margin: 0 auto;
  display: flex;
  justify-content: center;
  align-items: end;
  flex-flow: row wrap;
  max-width: 720px;
  height: 100%;
}

.game-screen-section {
  display: flex;
  flex: auto;
  min-width: 50%;
  flex-flow: column wrap;
}

#game-screen-input {
  min-width: 100%;
  align-self: start;
}

#game-note-display {
  align-items: center;
  justify-content: flex-end;
}

@media (orientation: landscape) {
  #game-note-display, #game-screen-input {
    min-height: 50%;
  }
}

@media (orientation: portrait) {
  #game-note-display, #game-screen-input {
    min-height: 33%;
    min-width: 100%;
  }
}

button.quit {
  padding: 5px;
  font-weight: bold;
  border: none;
  outline: none;
  background-color: lightblue;
  cursor: pointer;
}

button.quit:hover {
  background-color: steelblue;
}

button.quit:focus {
  outline: none;
  background-color: lightblue;
}

button.quit:active {
  background-color: yellow;
}
</style>

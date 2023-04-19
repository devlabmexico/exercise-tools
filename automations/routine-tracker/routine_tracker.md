<!--link rel="stylesheet" href="https://unpkg.com/sakura.css/css/sakura.css" type="text/css"-->
<div id="root"></div>

<span id="data" class="hide">
{routine}
</span>


<script type="module">
  import { createContext, Component, Fragment, h, render} from 'https://esm.sh/preact';
  import htm from 'https://esm.sh/htm';
  import { useContext, useEffect, useState,  useReducer} from 'https://esm.sh/preact/hooks';

  const STANDARD_REST_TIME = 2000;
  const Timer = function(callback, delay) {
      var timerId, start, remaining = delay;
  
      this.pause = function() {
          window.clearTimeout(timerId);
          timerId = null;
          remaining -= Date.now() - start;
      };
  
      this.resume = function() {
          if (timerId) {
              return;
          }
  
          start = Date.now();
          timerId = window.setTimeout(callback, remaining);
      };
  
      this.start = this.resume;
  };
  
  const StateAccessor = createContext('light');


  // Initialize htm with Preact
  const html = htm.bind(h);

  // Setup initial data
  const ROUTINE_SET = 'SET_ROUTINE'
  
  // Lifecycle routine
  const ROUTINE_INITIAL_STATE = 'ROUTINE_INITIAL_STATE'
  const ROUTINE_PERFORM = 'PERFORM_ROUTINE'
  const ROUTINE_FINISH = 'FINISH_ROUTINE'
  
  // Lifecycle exercise
  const EXERCISE_SETUP = 'EXERCISE_SETUP'
  const EXERCISE_PERFORM = 'PERFORM_EXERCISE'
  const EXERCISE_PAUSE = 'PAUSE_EXERCISE'
  const EXERCISE_RESUME = 'RESUME_EXERCISE'
  const EXERCISE_PREVIOUS = 'PREVIOUS_EXERCISE'
  const EXERCISE_NEXT = 'NEXT_EXERCISE'
  const EXERCISE_REST_TIME = 'REST_TIME_EXERCISE'
  const EXERCISE_STOP = 'STOP_EXERCISE'
  const EXERCISE_FINISH = 'STOP_EXERCISE'

  const StartRoutine = ({dispatch, exercises}) => {
    let startAction = () => {
      dispatch({type: ROUTINE_PERFORM})
      dispatch({type: EXERCISE_SETUP, dispatch})
      dispatch({type: EXERCISE_PERFORM})
    }
    return html`<div id="start_routine">
      <button onClick=${startAction}>Start routine</button>
    </div>
    `
  }

  const PerformRoutine = ({dispatch, exercise}) => {
    let stageExercise = exercise.stage === EXERCISE_REST_TIME ? 
      html`<${RestExercise} dispatch=${dispatch} exercise=${exercise} />` : 
      html`<${PerformExercise} dispatch=${dispatch} exercise=${exercise} />`
    return html`<div id="perform_routine">
      ${stageExercise}
    </div>
    `
  }

  const EndRoutine = ({dispatch, exercises}) => {
    let restartAction = () => dispatch({type: ROUTINE_INITIAL_STATE})
    return html`<div id="end_routine">
      <h3>Congratulations!</h3>
      <button onClick=${restartAction}>Download Progress</button>
    </div>
    `
  }

  const RestExercise = ({dispatch, exercise}) => {
    let skipAction = () => dispatch({type: EXERCISE_PERFORM})

    return html`<div id="restExercise">
      <header>
        <strong>Next exercise</strong>
        <h3 className="exercise_name">${exercise.exercise.name}</h3>
      </header>
      <div className="exercise_actions">
        <button onClick=${skipAction}>Skip rest time</button>
      </div>
    </div>`
  }
  const PerformExercise = ({dispatch, exercise}) => {
    let endAction = () => {
      dispatch({type: EXERCISE_FINISH, dispatch})
      dispatch({type: ROUTINE_FINISH, dispatch})
    }
    let pauseAction = () => dispatch({type: EXERCISE_PAUSE})
    let resumeAction = () => dispatch({type: EXERCISE_RESUME})
    let previousExerciseAction = () => {
      dispatch({type: EXERCISE_PREVIOUS})
      dispatch({type: EXERCISE_SETUP, dispatch})
      dispatch({type: EXERCISE_PERFORM})
    }
    let nextExerciseAction = () => {
      dispatch({type: EXERCISE_FINISH, dispatch})
      dispatch({type: EXERCISE_NEXT, dispatch})
      dispatch({type: EXERCISE_SETUP, dispatch})
      dispatch({type: EXERCISE_REST_TIME, dispatch})
    }
    let recommendations = exercise.exercise.recommendations
    let time_by_round = recommendations.time_by_round != 0 ? recommendations.time_by_round : ""
    let repetitions = recommendations.repetitions != 0 ? html`<strong>${recommendations.repetitions}</strong> repetitions ` : ""
    let by_side = recommendations.by_side ? html`<strong>each side</strong> ` : ""
    let relax_by_rep = recommendations.relax_by_rep != 0 ? recommendations.relax_by_rep : ""
    let series = recommendations.series != 0 ? html`x <strong>${recommendations.series}</strong>` : ""
    let pauseComponent = exercise.stage !== EXERCISE_PAUSE ?
        html`<button onClick=${previousExerciseAction} disabled=${exercise.index === 0}>Previous exercise</button>
             <button onClick=${pauseAction}>Pause</button>
             <button onClick=${nextExerciseAction}>Next Exercise</button>
        ` :
        html`<button onClick=${resumeAction}>Resume</button>`
    return html`<div id="perform_exercise">
      <header>
        <h3 className="exercise_name">${exercise.exercise.name}</h3>
          ${repetitions}
          ${by_side}
          ${series}
      </header>
      <button onClick=${endAction}>Finish routine</button>
      <div className="exercise_img">
        <img src=${exercise.exercise.resources.image} />
      </div>  
      <div className="exercise_actions">
        ${pauseComponent}
      </div>
    </div>
    `
  }

  const reducer = (state, action) => {
    console.log(action.type)
    switch (action.type) {
      case EXERCISE_SETUP:
        if (state.currentExercise.index === undefined) return state;
        { 
          let currentExercise = {...state.currentExercise}
          currentExercise.stage = action.type
          currentExercise.exercise = state.exercises[currentExercise.index]

          let exerciseTime = currentExercise.exercise.recommendations.time_by_round
          if (exerciseTime > 0) {
            let timer = new Timer(() => {
              dispatch({type: EXERCISE_FINISH, dispatch})
              dispatch({type: EXERCISE_NEXT, dispatch}) 
              dispatch({type: EXERCISE_SETUP})
              dispatch({type: EXERCISE_REST_TIME, dispatch})
              
            }, exerciseTime)
            currentExercise.timer = action.timer;
          } 
          return {...state, currentExercise}
        }
      case EXERCISE_PERFORM:
        if (state.currentExercise.restTimer) state.currentExercise.restTimer.pause();
        if (state.currentExercise.timer) state.currentExercise.timer.start();
        return {...state, currentExercise: {...state.currentExercise, restTimer: undefined, stage: action.type}}
      case EXERCISE_PAUSE:
        if (state.currentExercise.timer) currentExercise.timer.pause();
        return {...state, currentExercise: {...state.currentExercise, stage: action.type}}
      case EXERCISE_RESUME:
        if (state.currentExercise.timer) currentExercise.timer.resume();
        return {...state, currentExercise: {...state.currentExercise, stage: action.type}}
      // case EXERCISE_STOP:
      case EXERCISE_PREVIOUS:
        if (state.currentExercise.index === 0) return state
        {
          let currentExercise = {...state.currentExercise}
          if (currentExercise.timer) currentExercise.timer.pause();
          currentExercise.index -= 1
          currentExercise.timer = undefined
          currentExercise.stage = action.type
  
          return {...state, currentExercise: currentExercise}
        }
      case EXERCISE_NEXT:
        if (state.currentExercise.index === undefined || state.exercises.length <= state.currentExercise.index + 1) {
            console.log(action.dispatch({type: ROUTINE_FINISH}))
            return state;
        } {
          let currentExercise = {...state.currentExercise}
          currentExercise.index += 1
          currentExercise.timer = undefined
          currentExercise.stage = action.type
  
          return {...state, currentExercise: currentExercise}
        }
      case EXERCISE_REST_TIME:
        if (state.currentExercise.index === undefined) return state; 
        let timer = new Timer(() => {
          action.dispatch({type: EXERCISE_PERFORM})
        }, STANDARD_REST_TIME)
        timer.start()
        return {...state, currentExercise: {...state.currentExercise, restTimer: timer, stage: action.type}}
      case EXERCISE_FINISH: 
        let currentExercise = {...state.currentExercise}
        if (currentExercise.timer) currentExercise.timer.pause();
        // TODO: Consolidate exercise state

        return {...state, currentExercise: {...state.currentExercise, stage: action.type}}

      case ROUTINE_SET:
        return {...state, exercises: action.value}
      case ROUTINE_INITIAL_STATE:
        return {...state, currentExercise: {}, routineStage: action.type}
      case ROUTINE_PERFORM:
        return {...state, currentExercise: {index: 0}, routineStage: action.type}
      case ROUTINE_FINISH:
        return {...state, currentExercise: {}, routineStage: action.type}

      default: throw new Error('Unexpected action')
    }
  }

  const slicer = state => name => {
    return state[name]
  } 

  const initialState = {
    routineStage: ROUTINE_INITIAL_STATE, 
    exercises: [],
    currentExercise: {}
  }

  function App (props) {
    const [configuration, setConfiguration] = useState({})
    const [schedule, setSchedule] = useState({})
    const [routine, setRoutine] = useState({})
    const [state, dispatch] = useReducer(reducer, initialState);

    useEffect(() => {
      let raw_data = document.querySelector("#data").textContent;
      let exercises = JSON.parse(raw_data);
      setRoutine(exercises)
      dispatch({type: ROUTINE_SET, value: Object.values(exercises.exercises)});
    }, []);

    const component = state.routineStage == ROUTINE_INITIAL_STATE ? 
                         StartRoutine({dispatch, exercises: state.exercises}) :
                      state.routineStage == ROUTINE_FINISH ? 
                         EndRoutine({dispatch}) :
                         PerformRoutine({dispatch, exercises: state.exercises, exercise: state.currentExercise})
    return html`<${StateAccessor.Provider} value=${slicer(state)}>
      <div className="container">
        ${component}
      </div>
    </${StateAccessor.Provider}>`;
  }

  const root = document.querySelector("#root");
  (function loader() {
    if (document.querySelector("#data").textContent.trim() !== '') {
      render(html`<${App} name="Routine Tracker" />`, root);
      return;
    }
    setTimeout(loader, 500);
   })()
</script>
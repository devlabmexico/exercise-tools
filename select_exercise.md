<link rel="stylesheet" href="https://unpkg.com/sakura.css/css/sakura.css" type="text/css">


<div id="root"></div>

<pre id="initial_configuration" class="hide">
{
  "patient_name": "{patient_name}",
  "professional_name": "{health_professional_name}",
  "diagnosis": "{diagnosis}"
}
</pre>
<span id="data" class="hide">
{exercises}
</span>


<script type="module">
  import { h, Component, render} from 'https://esm.sh/preact';
  import htm from 'https://esm.sh/htm';
  import { useState, useEffect } from 'https://esm.sh/preact/hooks';

  // Initialize htm with Preact
  const html = htm.bind(h);

  function SelectedExercise (props)  {
    let exercises = Object.entries(props.exercises).map(([name, exercise]) => {
      let steps = exercise.steps.map(step => html`<li>${step}</li>`);
      let instructions = props.options.instructions ? html`<ul>${steps}</ul>` : ""  
      return html`<div className="cell border" onDblClick=${(event) => props.toggleExercise(name, exercise)}>
        <h3>${exercise.name}</h3>
        ${instructions}
        <div className="exercise-config">
        <fieldset> 
          <label>"time_by_round"</label>
          <input type=number value=60 name="Time by round" />
        </fieldset>
        <fieldset> 
          <label>"relax_by_rep"</label>
          <input type=number value=0 name="relax_by_rep" />
        </fieldset>
        <fieldset> 
          <label>"by_side"</label>
          <input type=checkbox  name="by_side" checked />
        </fieldset>
        <fieldset> 
          <label>"repetitions"</label>
          <input type=number  name="repetitions" value=10 />
        </fieldset>
        <fieldset> 
          <label>"series"</label>
          <input type=number  name="series" value=3 />
        </fieldset>
        <fieldset> 
          <label for="frequency">"frequency"</label>
          <select id="frequency">
            <option value="daily">daily</option>
          </select>
        </fieldset>
        <fieldset> 
          <label for="period">"period"</label>
          <select id="period">
            <option value="15 days">15 days</option>
          </select>
        </fieldset>
        </div>
      </div>`
    });
    return html`<div className="${props.className}">
       ${exercises}
    </div>
    `
  }

  function ExerciseFilters({area, effect, updateArea, updateEffect, exerciseArea, exerciseEffect}) {
    let areas = Object.keys(exerciseArea).map(areaName => 
                    html`<option value="${areaName}">${areaName}</option>`)

    let effectsValues = area !== "null" ? exerciseArea[area].values() : exerciseEffect.values();
    console.log(exerciseArea[area])
    let actions = [...effectsValues].map(effectName => 
                    html`<option value="${effectName}">${effectName}</option>`)

    let handleSelectArea = (event) => {
        updateArea(event.target.value);
        updateEffect("null");
    }

    return html`<div>
      <fieldset>
      <label for="body_part">Exercise Area</label>  
      <select value=${area} id="body_part" onInput=${handleSelectArea}>
        <option value="null">All</option>
        ${areas}
      </select>
      </fieldset>
      <fieldset>
      <label for="action">Exercise Effect</label>
      <select value=${effect} id="action" onInput=${(e) => updateEffect(event.target.value) }>
        <option value="null">All</option>
        ${actions}
      </select>
      </fieldset>
    </div>`;
  }  

  function ListExercises (props) {
    const [selectedArea, setSelectedArea] = useState("null"); 
    const [selectedAction, setSelectedAction] = useState("null");

    let exercises = Object.entries(props.exercises)
                          .filter(([name, exercise]) => {
                             let actions = Object.values(exercise['target']);                             
                             if (selectedArea === "null" && 
                                 selectedAction !== "null" && 
                                 actions.includes(selectedAction)) return true;
                             if (selectedArea !== "null" && exercise['target'][selectedArea] === undefined) return false;
                             if (selectedAction !== "null" && exercise['target'][selectedArea] !== selectedAction) return false;
                             
                             return true;
    })
                          .map(([name, exercise]) => {
      return html`<div className="no-margin exercise_list_item ${exercise.selected ? 'selected' : ''}" 
                       onClick=${(event) => props.toggleExercise(name, exercise)}>
        <p className="no-margin">${exercise.name}</p>
      </div>`
    });
    return html`<div className=${props.className}>
       <${ExerciseFilters} exerciseArea=${props.exerciseArea} 
                           exerciseEffect=${props.exerciseEffect}
                           area=${selectedArea}
                           effect=${selectedAction}
                           updateArea=${setSelectedArea}
                           updateEffect=${setSelectedAction}
       />
       ${exercises}
    </div>
    `
  }

  function PrintOptions ({options, setOptions}) {
    function handleUpdateOption(name, value) {
      setOptions({
          ...options, 
          [name]: value
      })
    }

    return html`
    <div className="options hide_on_print">
      <fieldset>
      <label for="orientation">Orientation</label>
      <select id="orientation" value=${options.orientation} onInput=${e => handleUpdateOption("orientation", e.target.value)}>
        <option value="portrait">Portrait</option>
        <option value="landscape">Landscape</option>
      </select>
      </fieldset><fieldset>
      <label for="columns">Number of columns</label>
      <select id="columns" value=${options.columns} onInput=${e => handleUpdateOption("columns", e.target.value)}>
        <option value="1">1</option>
        <option value="2">2</option>
        <option value="3">3</option>
        <option value="4">4</option> 
      </select>
      </fieldset><fieldset>
      <input type="checkbox" id="instructions" 
         name="instructions" 
         checked=${options.instructions === true} 
         onInput=${e => handleUpdateOption("instructions", e.target.checked)}
      />      
      <label for="instructions">Show Instructions</label>
      
      </fieldset>
      <a href="javascript:window.print()" class="hide_on_print btn">Print this Page</a>

    </div>`
  }

  function App (props) {
    const [configuration, setConfiguration] = useState({}); 
    const [exercises, setExercises] = useState({});
    const [selectedExercises, setSelectedExercises] = useState({});
    const [exerciseArea, setExerciseArea] = useState([])
    const [exerciseEffect, setExerciseEffect] = useState(new Set())
    const [options, setOptions] = useState({
      orientation: "portrait",
      columns: "1",
      instructions: true,
    }) 

    function toggleExercise(name, current_exercise) {
        if (selectedExercises[name] === undefined) {
          let newSelectedExercises = { ...selectedExercises, [name]: current_exercise};
          setSelectedExercises(newSelectedExercises);
          let newExercises = {...exercises, [name]: {...current_exercise, selected: true}}
          setExercises(newExercises);
          console.log("add new exercise")
          return;
        }

        delete selectedExercises[name]
        setSelectedExercises(selectedExercises);
        let newExercises = {...exercises, [name]: {...current_exercise, selected: false}}
        setExercises(newExercises);
        console.log("remove exercise")
    }

    useEffect(() => {
      let raw_data = document.querySelector("#data").textContent;
      let rawConfiguration = document.querySelector("#initial_configuration").textContent;
      let jsonList = JSON.parse(raw_data);
      let jsonConfiguration = JSON.parse(rawConfiguration);

      let exercises = Object.fromEntries(jsonList.map((exercise => [exercise.name, exercise])))
      let exerciseArea = {};

      jsonList.forEach(exercise => {
         for (let [bodyPart, effect] of Object.entries(exercise.target)) {
           if (exerciseArea[bodyPart] === undefined) { exerciseArea[bodyPart] = new Set(); }
           
           exerciseArea[bodyPart].add(effect);
           exerciseEffect.add(effect)
         } 
      });
      setExerciseArea(exerciseArea);
      setExercises(exercises);
      setConfiguration(jsonConfiguration);
    }, []);

    useEffect(() => {
      var head = document.head || document.getElementsByTagName('head')[0]
      var cssText = `@page { size: ${options.orientation}; }`
      var styleElement = document.querySelector('#print') || document.createElement('style')

      styleElement.type = 'text/css';
      styleElement.media = 'print';
      styleElement.id = 'print';

      if (styleElement.styleSheet){
        styleElement.styleSheet.cssText = cssText;
      } else {
        styleElement.textContent = cssText;
        head.appendChild(styleElement);
      }
    }, [options.orientation]);


    useEffect(() => {
      let r = document.querySelector(':root');
      r.style.setProperty('--exercises_rows', options.columns);
    }, [options.columns]);

    return html`<div>
      <${PrintOptions} setOptions=${setOptions} options=${options}/>

      <div className="container">
        <div className="pages">
        <div className="general">
          <h5>Exercise List</h5>
          <p className="no-margin"><strong>Health Professional</strong>: ${configuration.professional_name}</p>
          <p><strong>Patient</strong>: ${configuration.patient_name}</p>
          <hr />
          <p><strong>Notes</strong>: ${configuration.diagnosis}</p> 
          <hr />

        </div>
        <${SelectedExercise} className="selected_exercises" 
                             exercises=${selectedExercises} 
                             toggleExercise=${toggleExercise}
                             options=${options}
        />
        </div>
        <${ListExercises} className="exercises hide_on_print" exercises=${exercises} 
                          toggleExercise=${toggleExercise} 
                          exerciseArea=${exerciseArea}
                          exerciseEffect=${exerciseEffect}
        />
      </div>
    </div>`;
  }

  const root = document.querySelector("#root");
  render(html`<${App} name="World" />`, root);
</script>
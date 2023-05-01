<!--link rel="stylesheet" href="https://unpkg.com/sakura.css/css/sakura.css" type="text/css"-->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Roboto:ital,wght@0,100;0,300;0,400;0,500;0,700;0,900;1,100;1,300;1,400;1,500&display=swap" rel="stylesheet">

{affected_zone}
{exercise_type}
{severity}

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

  function SelectedExercise ({name, exercise, showInstructions, toggleExercise, updateRecommendations})  {
    let recommendations = exercise.recommendations;
    let steps = exercise.steps.map(step => html`<li>${step}</li>`);
    let instructions = showInstructions ? html`<ul>${steps}</ul>` : "";

    return html`<div className="cell border" onDblClick=${(event) => toggleExercise(name, exercise)}>
      <div>
        <h3>${exercise.name}</h3>
        <div className="instructions">
          <div className="image"><img src="${exercise.resources.image}" /></div>
          ${instructions}
        </div>
      </div>
      <div className="exercise-config hide_on_print">
        <fieldset className="fieldset_remove_decoration"> 
          <label for="repetitions">Repetitions per round</label>
          <input id="repetitions" min=0 max=100 type=number value=${recommendations.repetitions} 
                 onInput=${(e) => updateRecommendations(name, "repetitions", event.target.value) } />
        </fieldset>        
      <fieldset className="fieldset_remove_decoration"> 
        <label for="time_by_round" >Time by round (seconds)</label>
        <input id="time_by_round" min=0 max=6000 type=number value=${recommendations.time_by_round}  
                 onInput=${(e) => updateRecommendations(name, "time_by_round", event.target.value) } />
      </fieldset>
      <fieldset className="fieldset_remove_decoration"> 
        <label for="relax_by_rep">Seconds to relax between repetition</label>
        <input id="relax_by_rep" min=0 max=6000 type=number value=${recommendations.relax_by_rep}   
                 onInput=${(e) => updateRecommendations(name, "relax_by_rep", event.target.value) } />
      </fieldset>
      <fieldset className="fieldset_remove_decoration"> 
        <label for="by_side">Exercise by side</label>
        <input id="by_side" type=checkbox   checked=${recommendations.by_side}  
                 onInput=${(e) => updateRecommendations(name, "by_side", event.target.checked) } />
      </fieldset>
      <fieldset className="fieldset_remove_decoration"> 
        <label for="series">Total series</label>
        <input id="series" min=0 max=30 type=number value=${recommendations.series}  
                 onInput=${(e) => updateRecommendations(name, "series", event.target.value) } />
      </fieldset>
      <fieldset className="fieldset_remove_decoration"> 
        <label for="frequency">Frequency</label>
        <select id="frequency"  value=${recommendations.frequency}
                 onInput=${(e) => updateRecommendations(name, "frequency", event.target.value) } >
          <option value="daily">daily</option>
        </select>
      </fieldset>
      <fieldset className="fieldset_remove_decoration"> 
        <label for="period">Period</label>
        <select id="period"  value="${recommendations.period}"
                 onInput=${(e) => updateRecommendations(name, "period", event.target.value) } >
          <option value="15 days">15 days</option>
        </select>
      </fieldset>
      </div>
      <div className="recommendations">
        <p>Series: ${recommendations.series} 
                   ${recommendations.relax_by_rep ? 
                       " with " + recommendations.relax_by_rep + " seconds to relax after each repetition." : ""}
        </p>          
        <p>Repetitions: ${recommendations.repetitions}  
                        ${recommendations.by_side ? " each side " : ""} 
                        ${recommendations.time_by_round ? 
                            " in " + recommendations.time_by_round + " seconds" : ""}
        </p>
        <p>Frequency: <div style="display: inline-block; text-transform: capitalize;">${recommendations.frequency}</div> by a period of ${recommendations.period}</p>
      </div>
    </div>`
  
  }
  function SelectedExercises (props)  {
    let exercises = Object.entries(props.exercises).map(([name, exercise]) => 
                      html`<${SelectedExercise} name=${name} 
                                            exercise=${exercise}
                                            updateRecommendations=${props.updateRecommendations}
                                            showInstructions=${props.options.instructions} 
                                            toggleExercise=${props.toggleExercise} />`);
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
      <fieldset className="fieldset_remove_decoration">
      <label for="body_part">Exercise Area</label>  
      <select value=${area} id="body_part" onInput=${handleSelectArea}>
        <option value="null">All</option>
        ${areas}
      </select>
      </fieldset>
      <fieldset className="fieldset_remove_decoration">
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
    const [showFilters, setShowFilters] = useState(false);

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

    let exercisesSection = showFilters ? html`<div className=${props.className}>
       <${ExerciseFilters} exerciseArea=${props.exerciseArea} 
                           exerciseEffect=${props.exerciseEffect}
                           area=${selectedArea}
                           effect=${selectedAction}
                           updateArea=${setSelectedArea}
                           updateEffect=${setSelectedAction}
       />
       ${exercises}</div>
    `: ''
    return html`<div>
       ${exercisesSection}
       <button className="exercise-toggle btn-action" onClick=${() => setShowFilters(!showFilters)}>
         Add/Remove Exercises
       </button> 
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
      <fieldset className="fieldset_remove_decoration">
      <label for="orientation">Orientation</label>
      <select id="orientation" value=${options.orientation} onInput=${e => handleUpdateOption("orientation", e.target.value)}>
        <option value="portrait">Portrait</option>
        <option value="landscape">Landscape</option>
      </select>
      </fieldset><fieldset className="fieldset_remove_decoration">
      <label for="columns">Number of columns</label>
      <select id="columns" value=${options.columns} onInput=${e => handleUpdateOption("columns", e.target.value)}>
        <option value="1">1</option>
        <option value="2">2</option>
        <option value="3">3</option>
        <option value="4">4</option> 
      </select>
      </fieldset><fieldset className="fieldset_remove_decoration">
      <input type="checkbox" id="instructions" 
         name="instructions" 
         checked=${options.instructions === true} 
         onInput=${e => handleUpdateOption("instructions", e.target.checked)}
      />      
      <label for="instructions">Show Instructions</label>
      
      </fieldset>
    </div>`
  }

  function App (props) {
    const [configuration, setConfiguration] = useState({}) 
    const [exercises, setExercises] = useState({})
    const [selectedExercises, setSelectedExercises] = useState({})
    const [exerciseArea, setExerciseArea] = useState([])
    const [exerciseEffect, setExerciseEffect] = useState(new Set())
    const [notes, setNotes] = useState("")
    const [options, setOptions] = useState({
      orientation: "portrait",
      columns: "1",
      instructions: true,
    }) 

    let toggleExercise = (name, current_exercise) => {
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

    let updateRecommendations = (exerciseName, fieldName, newValue) => {
      let exercise = selectedExercises[exerciseName];
      setSelectedExercises({
        ...selectedExercises, 
        [exerciseName]: {
          ...exercise, 
          recommendations: {
            ...exercise.recommendations, 
            [fieldName]: newValue
          }
        }
      })

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
      setNotes(jsonConfiguration.diagnosis);
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

    function downloadRoutine() {
      if (Object.keys(selectedExercises).length == 0) {
        alert("Nothing to download");
        return;
      }

      let routine = { 
        configuration,
        notes,
        exercises: selectedExercises,
        options 
      }

      let text = JSON.stringify(routine)
      let objectDate = new Date();
      let day = objectDate.getDate();
      let month = objectDate.getMonth();
      let year = objectDate.getFullYear();
      let filename = `${configuration.patient_name.replaceAll(' ', '_')}-${month}-${day}-${year}.json`
      let element = document.createElement('a');
      element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
      element.setAttribute('download', filename);

      element.style.display = 'none';
      document.body.appendChild(element);

      element.click();

      document.body.removeChild(element);
    }

    return html`<div>
      <${PrintOptions} setOptions=${setOptions} options=${options}/>
      <button onClick=${() => window.print()} class="hide_on_print btn btn-primary">Print Routine</a>
      <button onClick=${() => downloadRoutine()} class="hide_on_print btn">Download Routine</a>

      <div className="container">
        <div className="pages">
        <div className="general">
          <p className="no-margin"><strong>Health Professional</strong>: ${configuration.professional_name}</p>
          <p><strong>Patient</strong>: ${configuration.patient_name}</p>
          <hr />
          <fieldset className="align_baseline_elements remove_border hide_on_print">
            <label className="hide_on_print" for="notesEditor">Notes</strong>
            <textarea className="resize_vertical hide_on_print" id="notesEditor" value=${notes} cols=180 rows=3 oninput=${(e) => setNotes(event.target.value) } />
          </fieldset>
          <p className="notes"><strong>Notes</strong>: <pre style="word-wrap: break-word; white-space: pre-wrap;">${notes}</pre></p> 
          <hr />

        </div>
        <${SelectedExercises} className="selected_exercises" 
                             exercises=${selectedExercises} 
                             toggleExercise=${toggleExercise}
                             options=${options}
                             updateRecommendations=${updateRecommendations}
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
  (function loader() {
    if (document.querySelector("#data").textContent.trim() !== '') {
      render(html`<${App} name="World" />`, root);
      return;
    }
    setTimeout(loader, 500);
   })()
</script>
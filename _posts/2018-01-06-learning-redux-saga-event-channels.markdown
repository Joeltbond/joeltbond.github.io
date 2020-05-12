---
layout: post
title: "Learning Redux Saga: Event Channels with Web MIDI"
---

_This post was originally published on [Medium](https://medium.com/@joeltbond/learning-redux-saga-event-channels-29dee438fd7b)_

Recently we started using Redux Saga at work to handle the side-effects of our React/Redux app. For the most part we’ve been using it in a pretty straightforward way - watching with `takeEvery` and then making api calls with call or apply effects and eventually getting the data into the store with put. But I know that Saga is much more powerful than just that. I wanted to take some time to look into some of the more advanced functionality. In this post we will look at event channels.

Without channels, Saga provides a straightforward way of making asynchronous calls and waiting for a response from that specific call. But what if we want to listen to an unspecified number of events from source outside of our React components? Saga’s event channels make this possible.

## An example

The typical example for using an event channel is listening to messages over a web socket. But let’s try something a little more interesting. Say we are creating a web audio synthesizer or some other musical app that needs to listen to incoming MIDI events and display some values about those events.

One way to accomplish this would be to simply set up the listener in the UI. You could start listening for midi events on componentDidMount and either keep track of the note values in local component state or dispatch actions with payloads that eventually make their way back to the component as props, possibly after some processing.

```ts
import React, { Component } from "react";
import { onmidimessage } from "my-action-creators-file";

class MidiDisplay extends Component {
  componentDidMount() {
    const midiAccess = window.navigator.requestMIDIAccess({
      sysex: false,
    }); // loop through inputs
    // assign onmidimessage action creator to each
    // etc...
  }
  render() {
    // display notes
  }
}

const mapStateToProps = (state) => ({
  notes: state.app.notes,
});

export default connect(mapStateToProps, { onmidimessage })(MidiDisplay);
```

But this makes our UI component brittle and difficult to test. It’s a lot of specific Web MIDI implementation code mixed with our UI code, especially if we want to do any sort of processing of the messages. We want to keep our components (even our connected containers) as light and dumb as we can. The component displaying the information should be concerned with just that, display:

```ts
import React from "react";

export default ({ notes }) => <div>Notes: {Object.keys(notes).join(", ")}</div>;
```

Here we have component that takes a dictionary where the note values are the keys and values are the velocity at which the note was played. For now we will simply display the notes that are currently being played, separated by commas.

We’ll then wrap this component in a container that gets the note/velocity dictionary from the store and hands it down to this component as a prop:

```ts
import React, { Component } from "react";
import { connect } from "react-redux";
import { appMounted } from "my-action-creators-file";
import MidiDisplay from "../components/MidiDisplay";

class App extends Component {
  componentDidMount() {
    this.props.appMounted();
  }

  render = () => <MidiDisplay notes={this.props.notes} />;
}

const mapStateToProps = (state) => ({
  notes: state.app.notes,
});

export default connect(mapStateToProps, { appMounted })(App);
```

The other thing that happens here is that when our app loads up we fire an action that our saga will use to do the MIDI initialization. Let’s look at some saga code:

```ts
function* appMountedSaga() {
  if (window.navigator.requestMIDIAccess) {
    try {
      const midiAccess = yield call(
        [window.navigator, window.navigator.requestMIDIAccess],
        {
          sysex: false,
        }
      );
      yield call(onMidiSuccess, midiAccess);
    } catch (error) {
      yield put({ type: MIDI_UNSUPPORTED });
      console.error(error);
    }
  }
}

export const appSagas = [takeEvery(APP_MOUNTED, appMountedSaga)];
```

The `APP_MOUNTED` action from our container triggers the `appMountedSaga`. In this file we just export an array of calls to `takeEvery` (just one here). I’ve gotten into the habit of exporting sagas as arrays. This way I can combine them into a rootSaga without caring how many each domain is actually exporting:

```ts
export function* rootSaga() {
  yield all([...appSagas, ...otherSagas, ...otherOtherSagas]);
}
```

`appMountedSaga` requests MIDI access. If the browser doesn’t support MIDI or if there is any error getting access we will add some errors to the store which we may want to display later to the user. Otherwise we call onMidiSuccess.

```ts
function* onMidiSuccess(midiAccess) {
  const channel = yield call(createMidiEventChannel, midiAccess);

  while (true) {
    const message = yield take(channel);
    yield call(onmidimessage, message);
  }
}
```

Here we create our event channel and create an endless loop to take every event that is emitted from it. Remember that since we are in a generator, this is not going to blow up. For each iteration, we yield until take receives a value from the channel. The message that is emitted gets passed to `onmidimessage` which will do some processing before putting and action to the store. Let’s look at the function that creates the channel:

```ts
function createMidiEventChannel(midiAccess) {
  return eventChannel((emitter) => {
    const inputs = midiAccess.inputs.values();
    for (
      let input = inputs.next();
      input && !input.done;
      input = inputs.next()
    ) {
      // each time there is a MIDI message call the onmidimessage
      // function
      input.value.onmidimessage = emitter;
    } // The subscriber must return an unsubscribe function. We'll
    // just return no op for this example.
    return () => {
      // Cleanup event listeners. Clear timers etc...
    };
  });
}
```

Here we return a new channel via `eventChannel()` from Redux Saga. We pass in a callback that takes an emitter functions and describes when and how it should be called. `midiAccess.inputs.values()` returns an iterator that we can use to loop through all available MIDI inputs and add a listener for messages. We’ll set our emitter callback as the onmidimessage method for each input. Note that you need to return an unsubscribe function so that the channel can clean up after itself when it is closed. For this example we will leave it as a no op since we want to keep the channel open.

Here is the entire redux module, including the reducer and actions:

```ts
import { eventChannel } from "redux-saga";
import { takeEvery, take, call, put } from "redux-saga/effects";

const APP_MOUNTED = "app/APP_MOUNTED";
const MIDI_UNSUPPORTED = "app/MIDI_UNSUPPORTED";
const MIDI_ACCESS_FAILURE = "app/MIDI_ACCESS_FAILURE";
const MIDI_NOTE_ON = "app/MIDI_NOTE_ON";
const MIDI_NOTE_OFF = "app/MIDI_NOTE_OFF";

const initialState = {
  errors: [],
  notes: {},
};

export default (state = initialState, action = {}) => {
  switch (action.type) {
    case MIDI_UNSUPPORTED:
      return {
        ...state,
        errors: state.errors.concat(
          "Midi input is not supported in your browser"
        ),
      };
    case MIDI_ACCESS_FAILURE:
      return {
        ...state,
        errors: state.errors.concat("Failed to access MIDI"),
      };
    case MIDI_NOTE_ON: {
      return {
        ...state,
        notes: {
          ...state.notes,
          [action.payload.note]: action.payload.velocity,
        },
      };
    }
    case MIDI_NOTE_OFF: {
      const newNotes = { ...state.notes };
      delete newNotes[action.payload.note];
      return {
        ...state,
        notes: newNotes,
      };
    }
    default:
      return state;
  }
};

export const appMounted = () => ({
  type: APP_MOUNTED,
});

function* onmidimessage({ data }) {
  const [type, note, velocity] = data;

  switch (type) {
    case 144:
      yield put({ type: MIDI_NOTE_ON, payload: { note, velocity } });
      break;
    case 128:
      yield put({ type: MIDI_NOTE_OFF, payload: { note } });
      break;
    default:
  }
}

function createMidiEventChannel(midiAccess) {
  return eventChannel((emitter) => {
    const inputs = midiAccess.inputs.values();
    for (
      var input = inputs.next();
      input && !input.done;
      input = inputs.next()
    ) {
      // each time there is a midi message call the onmidimessage function
      input.value.onmidimessage = emitter;

      // the emitter can also be called with the 'DONE' constant from redux-saga
      // which causes the channel to close. For this example we will never close.
    }

    // The subscriber must return an unsubscribe function. We'll just return no op for this example
    return () => {
      // Cleanup event listeners. Clear timers etc...
    };
  });
}

function* onMidiSuccess(midiAccess) {
  const channel = yield call(createMidiEventChannel, midiAccess);

  while (true) {
    const message = yield take(channel);
    yield call(onmidimessage, message);
  }
}

function* appMountedSaga() {
  if (window.navigator.requestMIDIAccess) {
    try {
      const midiAccess = yield call(
        [window.navigator, window.navigator.requestMIDIAccess],
        {
          sysex: false,
        }
      );
      yield call(onMidiSuccess, midiAccess);
    } catch (error) {
      yield put({ type: MIDI_ACCESS_FAILURE });
      console.error(error);
    }
  } else {
    yield put({ type: MIDI_UNSUPPORTED });
  }
}

export const appSagas = [takeEvery(APP_MOUNTED, appMountedSaga)];
```

## That’s it!

I hope you find this example helpful. I tend to think of Redux middleware as something to handle network calls but it’s useful for any sort of I/O would make your UI code less declarative. By moving the MIDI messaging code out of the UI code we were able to clean things up and Redux Saga event channels helped us do it.

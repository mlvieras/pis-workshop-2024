# React Workshop

## Setting up the project

### Install the appropriate versions of Node and NPM

There's a multitude of ways to install and manage Node versions. Usually, projects will require you to be able to spin up many versions of Node. It's simply too annoying to uninstall and install a different version each time. That's why there exist different version managers that support Node. Some of them are `nodenv`, `nvm` and `asdf`.

At Xmartlabs we've mostly used [nodenv](https://github.com/nodenv/nodenv), so these next instructions take for granted that you have it installed on your machine.

```sh
nodenv install 18.15.0
nodenv rehash
npm i -g npm@9.6.2
```

### Cloning the template

First step is cloning Xmartlabs' Create React App template.

* Go to [the Github repo](https://github.com/xmartlabs/react-template-xmartlabs)
* Run `degit`: `npx degit@latest xmartlabs/react-template-xmartlabs frontend`

### Run the newly-created project

```sh
cd frontend
npm install
npm start
```

## Let's start doing stuff!

We're going to create a very small application that lists some tasks the user has to do.

### No backend? No worries!

In order to quickly have a "backend" running, we're going to use [`json-server`](https://github.com/typicode/json-server). First, install the library with:

```sh
npm i -D json-server
```

Then, create a `db.json` file on the root of the react workshop:

```sh
touch db.json
```

Now, add sample data inside:

```json
{
  "messages": [
    {
      "id": 1,
      "content": "Don't forget: wash your clothes",
      "due_date": "2022-08-19T18:38:58.278Z",
      "is_complete": false
    },
    {
      "id": 2,
      "content": "Study for my exam",
      "due_date": "2022-08-20T10:00:58.278Z",
      "is_complete": true
    },
    {
      "id": 3,
      "content": "Take out the trash",
      "due_date": "2022-05-03T09:00:00.278Z",
      "is_complete": false
    },
    {
      "id": 4,
      "content": "Take out the trash",
      "due_date": "2022-03-01T18:38:58.278Z",
      "is_complete": false
    }
  ]
}
```

Let's now start our server:

```sh
npx json-server --watch db.json -p 3001
```

It should output something like:

```text
  \{^_^}/ hi!

  Loading db.json
  Done

  Resources
  http://localhost:3001/messages

  Home
  http://localhost:3001

  Type s + enter at any time to create a snapshot of the database
  Watching...
```

### Fetching the messages

We'll need to create two things:

* The controller on the frontend that will handle the request.
* The serializer that will process the data.

Here's the controller:

```js
// src/networking/controllers/message-controller.ts
import { MessageSerializer } from 'networking/serializers/message-serializer';
import { ApiService } from 'networking/api-service';
import { API_ROUTES } from 'networking/api-routes';

class MessageController {
  static async getMessages() : Promise<Message[]> {
    const response = await ApiService.get<RawMessage[]>(API_ROUTES.MESSAGES);
    return (response.data || []).map(MessageSerializer.deSerialize);
  }
}

export { MessageController };
```

The serializer:

```js
// src/networking/serializers/message-serializer.ts
class MessageSerializer {
  static deSerialize(data: RawMessage) : Message {
    return {
      id: data.id,
      content: data.content,
      dueDate: new Date(data.due_date),
      isComplete: data.is_complete,
    };
  }
}

export { MessageSerializer };
```

And lastly, the types:

```ts
// src/networking/types/message.d.ts
type RawMessage = {
  id: number,
  content: string,
  due_date: string,
  is_complete: boolean,
};

type Message = {
  id: number,
  content: string,
  dueDate: Date,
  isComplete: boolean,
};
```

### Fetch our data!

(This will be done on the workshop itself).

* Update the environment variable that stores the backend URL.
* Add a `useEffect` call to fetch messages on page load.
* Add a `useState` call to store fetched messages local to the component.
* Render the information of the messages on screen.

### Create new messages!

* Add a form to create a new message.
* Create new types for "pending" messages.
* Add a method to the controller to create a message.
* Add a method to the serializer to serialize the message.
* When a message has been created, automatically add it to the list.

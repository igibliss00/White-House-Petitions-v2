# White House Petition v2

This app is to demonstrate the use of threading in the foreground and the background through Grand Central Dispatch and performSelector()

<img src="https://github.com/igibliss00/WhitehousePetitions/blob/master/README_assets/1.png" width="400">

This app is to demonstrate the Grand Central Dispatch

## Installing

```
git clone https://www.github.com/igibliss00/white-house-petitions-v2.git
```

## Features

### Dispatch Queues

GCD creates the queue for different tasks to be completed and depending on the QoS that you assign, the task will be prioritized.  

They are consisted of following:

- Main queue
- Background queue
    - User Interactive
    - User Initiated
    - The Utility queue
    - The background queue
    - The default queue: prioritized between user-initiated and utility  

### Thread Problems

There is a cardinal rule in using the dispatch queue is that you never want the user interface work to be done in the background.  This means when we assign certain tasks to the background threads, we have to make sure that all the subtasks within those tasks have to be assigned to the main thread.

The project initially had the following data fetch in the main thread, which locks up the entire app:

```
if let url = URL(string: urlString) {
    if let data = try? Data(contentsOf: url) {
        self.parse(json: data)
        return
    }
}

func parse(json: Data) {
    let decoder = JSONDecoder()
    if let jsonPetitions = try? decoder.decode(Petitions.self, from: json) {
        petitions = jsonPetitions.results
        tableView.reloadData()
    }
}
```

First, the Data(contentsOf: ) method is sent to the background thread:

```
DispatchQueue.global(qos: .userInitiated).async {
    if let url = URL(string: urlString) {
        if let data = try? Data(contentsOf: url) {
            self.parse(json: data)
            return
        }
    }
}
```

However, the parse(json: data) has the UI element of reloading the table view, which means that particular portion has to be brought forward to the main thread:

```
func parse(json: Data) {
    let decoder = JSONDecoder()
    if let jsonPetitions = try? decoder.decode(Petitions.self, from: json) {
        petitions = jsonPetitions.results
        DispatchQueue.main.async {
            self.tableView.reloadData()
        }
    }
}
```

### performSelector()

Instead of using the async() method of Dispatch.queue or Dispatch.global, which requires closure capturing, you can use the performSelector() method to do the same task.

For example the parse() function shown above could re-written as:

```
    func parse(json: Data) {
        let decoder = JSONDecoder()
        if let jsonPetitions = try? decoder.decode(Petitions.self, from: json) {
            petitions = jsonPetitions.results
            tableView.performSelector(onMainThread: #selector(UITableView.reloadData), with: nil, waitUntilDone: false)
        } else {
            performSelector(onMainThread: #selector(showError), with: nil, waitUntilDone: false)
        }
    }

    @objc func showError() {
        let ac = UIAlertController(title: "Loading error", message: "There was a problem loading the feed; please check your connection and try again.", preferredStyle: .alert)
        ac.addAction(UIAlertAction(title: "OK", style: .default, handler: nil))
        present(ac, animated: true)
    }
```

The important thing is to specify the functions with @objc since we’re using #selector

## Handle-connection, check response
Here I will analyze the function line-by-line and explain its use.
*  `fn handle_connection(mut stream: TcpStream) { ... }:`, This function takes a mutable reference to a TcpStream as its argument. This function would be used to handle incoming TCP connection requests.
*  `let buf_reader = BufReader::new(&mut stream);` here I declared a `BufReader` Object variable named `buf_reader` that wraps the mutable reference to the `TCP stream`. This will be used to read lines from the stream.
*  `stream.read(&mut buffer).unwrap();`
*  `let http_request: Vec<_> = buff_reader` Here I declared a vector variable named `http_request` that will process the `HTTP` request using the value from `buff_reader`. Then the `buff_reader` is furtherly processed, first to `.lines()` this will create an iterator over the lines, each iteration would have a `result` with a response of either `ok` or `err` depending on the successful connection or not. Then the `result` is passed on to `.map(|result| result.unwrap())`, this will extract the values of `ok` from the `result`. Next, `.take_while(|line| !line.is_empty())` will be responsible to keep taking the lines until it's false, by that it means to keep taking lines until the end of the `HTTP` request headers. Then `.collects()` will collect the previous result and store it again in the vector variable.
*  ` println!("Request: {:#?}", http_request);` Finally, this will be used for printing out the `HTTP` request vector from before. Using the `:#?` debug format for better clarity.
<br>
In a nutshell, this function will serve as the server for handling client requests.

##  Returning HTML
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/2c978af3-ce10-45f3-a9f2-bd4f11ffa213)
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/67a3b1b8-aec4-436f-b19a-1f0e8a48468c)

## Validating requests and selectively responding
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/a82adb24-e247-455d-bf52-5603aeea84ac)
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/843e3f15-9009-43a5-9002-9ca1b4b18795)
So in the code, the process of reading the request is further split into a few parts.
<br>

```
let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();
        let length = contents.len();`

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
```
Here, for processing the response I used a few if statements. If the request processes the root or `/` of the server hence it will direct to the `hello.html`. If they are not accessing the root then they will be directed to a different html namely `404.html`. However, there are a few repetitions of if and else here as they are responsible for reading and writing contents onto the stream, hence why we need to refactor the code to make it more concise.

#### After Refactoring


```
let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };`

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
```
The code that was previously duplicated has been moved outside of the if and else blocks and utilizes the use of the status_line and filename variables. This makes the differences between the two circumstances easier to understand and suggests that we only need to change one place in the code to change how the file reading and response writing behave. 

## Simulation of slow request
The difference between using `/sleep` and `/` normally, is that using `/sleep` simulates when there is a big amount of load on the server. Hence, in the case of `/sleep`, a sleep function is used on the thread module. This will make the loading time to access the `html` is slightly longer compared to accessing the root or `/`

##  Multithreaded server using Threadpool

1. `struct ThreadPool`, This struct represents a thread pool.
* {workers: Vec<Worker>} Contains a vector of workers, each in charge of carrying out assignments.
*  Option<mpsc::Sender<Job>>} as the sender It keeps track of an optional sender for the channel that employees utilize to receive tasks.
2. `Job Type Alias`, This is a type alias for the type of tasks that can be sent to the workers. It represents a closure that takes no arguments and returns nothing `(FnOnce())`, can be sent between threads `(Send)`, and lives for the `'static` lifetime.

3. `impl ThreadPool`
*  `new(size: usize) -> ThreadPool`, This constructor can be used to create a new ThreadPool with a certain size, or a number of workers.
  It initializes a channel `(mpsc::channel())`, to convey tasks to workers from the main thread.
  It creates a shared ownership (Arc) of a mutex-protected receiver to distribute among workers.
  The number of workers is initialized, and each has a cloned receiver and a unique ID.
  `execute<F>(&self, f: F) where F: FnOnce() + Send + 'static`, This method enqueues a task `(FnOnce())` into the thread pool for execution.
It wraps the task into a box and sends it through the channel to be picked up by an available worker.

4. `Drop` implementation for ThreadPool:
*  This implementation ensures graceful shutdown of the thread pool when it goes out of scope.
*  Workers are informed that no more tasks will be sent when it drop the sender.
*  It waits for each worker thread to finish (join) before exiting.
  
5. struct `Worker`:

* This struct represents a worker thread in the pool.

* It holds an ID and an optional handle to the spawned thread.

* impl `Worker`:

   `new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker`, This is a constructor for creating a new Worker.
   It starts a new thread that takes tasks from the shared receiver and runs it endlessly.
   The worker is terminated if an error occurs during the receiving process, signaling that the sender has been dropped.



  

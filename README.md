## Handle-connection, check response
Here I will analyze the function line-by line and explain it's use.
*  `fn handle_connection(mut stream: TcpStream) { ... }:`, This function takes a mutable reference to a TcpStream as its argument. This function would be used to handle incoming TCP connection requests.
*  `let buf_reader = BufReader::new(&mut stream);` here I declared a `BufReader` Object variable named `buf_reader` that wraps the mutable reference to the `TCPstream`. This will be used to read lines from the stream.
*  `stream.read(&mut buffer).unwrap();`
*  `let http_request: Vec<_> = buff_reader` here I declared a vector variable named `http_request` that will process the `http` request using the value from `buff_reader`. Then the `buff_reader` is furtherly processed, first to `.lines()` this will create an iterater over the lines, each iteration would have a `result` with a response of either `ok` or `err` depending on the successful connection or not. Then the `result` is passed on to `.map(|result| result.unwrap())`, this will extract the values of `ok` from the `result`. Next, `.take_while(|line| !line.is_empty())` will be responsible to keep taking the lines until it's false, by that it means to keep taking lines until the end of the `HTTP` request headers. Then `.collects()` will collect the previous result and store it again in the vector variable.
*  ` println!("Request: {:#?}", http_request);` Finally this will be used for printing out the `HTTP` request vector from before. Using the `:#?` debug format for better clarity.
In a nutshell this function will serve as the server for handling client requests.

##  Returning HTML
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/2c978af3-ce10-45f3-a9f2-bd4f11ffa213)
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/67a3b1b8-aec4-436f-b19a-1f0e8a48468c)

## Validating request and selectively responding
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/a82adb24-e247-455d-bf52-5603aeea84ac)
![image](https://github.com/Alvinzhafif/advprog-module6/assets/143392835/843e3f15-9009-43a5-9002-9ca1b4b18795)
So int the code the process of reading the request is furtherly splitted into few parts.
`
let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
`
Here, for processing the response I used a few if statements. If the request process the root or `/` of the server hence it will direct to the `hello.html`. If they are not accessing the root then they will be directed to a different html namely `404.html`. However, there is a few repetition of if and elses here as they are responsible for reading and writing contents onto the stream, hence why we need to refactor the code to make it more concise.
#### After Refactoring
`
let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
`
The code that was previously duplicated has been moved outside of the if and else blocks and utilizes the use of the status_line and filename variables. This makes the differences between the two circumstances easier to understand and suggests that we only need to change one place in the code to change how the file reading and response writing behave. 

## Simulation of slow request
The difference between using `/sleep` and `/` normally, is that using `/sleep` simulates when there is a big amount of load on the server. Hence, in the case of `/sleep`, a sleep function is used on the thread module. This will make the loading time to access the `html` is slightly longer compared to accessing the root or `/`

##  Multithreaded server using Threadpool

1. `struct ThreadPool`: This struct represents a thread pool.
*  `workers: Vec<Worker>` It holds a vector of workers, each responsible for executing tasks.
*  `sender: Option<mpsc::Sender<Job>>` It stores an optional sender for the channel used to communicate tasks to workers.
2. `Job Type Alias`, This is a type alias for the type of tasks that can be sent to the workers. It represents a closure that takes no arguments and returns nothing (FnOnce()), can be sent between threads (Send), and lives for the 'static lifetime.

3. `impl ThreadPool`
*  `new(size: usize) -> ThreadPool`, This is a constructor for creating a new ThreadPool with a specified number of workers (size).
** It initializes a channel `(mpsc::channel())` to communicate tasks between the main thread and workers.
** It creates a shared ownership (Arc) of a mutex-protected receiver to distribute among workers.
** It initializes size number of workers, each with a unique ID and a cloned receiver.
*  `execute<F>(&self, f: F) where F: FnOnce() + Send + 'static`, This method enqueues a task (FnOnce()) into the thread pool for execution.
It wraps the task into a box and sends it through the channel to be picked up by an available worker.

4. Drop implementation for ThreadPool:
*  This implementation ensures graceful shutdown of the thread pool when it goes out of scope.
*  It drops the sender, signaling to workers that no more tasks will be sent.
*  It waits for each worker thread to finish (join) before exiting.
  
5. struct `Worker`:

* This struct represents a worker thread in the pool.

* It holds an ID and an optional handle to the spawned thread.

* impl Worker:

** new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker: This is a constructor for creating a new Worker.
** It spawns a new thread that loops indefinitely, receiving and executing tasks from the shared receiver.
** If an error occurs during receiving (indicating the sender has been dropped), it breaks out of the loop, terminating the worker.



  

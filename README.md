<section id="distributed-systems-large-assignment-2" class="content">
<h1>Distributed Systems Large Assignment 2</h1>
<h3 id="atomic-disk-drive">Atomic Disk Drive</h3>
<p>Your task is to implement a distributed block device which stores data in a distributed register. The solution shall take the form of a Rust library. A template, public tests and additional files are provided in <a href="./dsassignment2.tgz">this package</a>.</p>
<h3 id="background">Background</h3>
<p>On UNIX-like operating systems, a <strong>block device</strong> is, simplifying, a device which serves random-access reads and writes of data in portions called <strong>blocks</strong>. Arguably, the most common types of block devices are disk drives (e.g., HDD and SSD). However, the term is also applied to various abstractions which implement the interface of the block device (and thus, from a user’s perspective, can be used like any other block device), but do not store data on physical devices. For instance, <strong>Atomic Disc Drive</strong> which stores data in a <strong>distributed register</strong>.</p>
<p>A distributed register consists of multiple <strong>processes</strong> (entities of the distributed register, not operating system processes) running in user space, possibly on multiple physical (or virtual) machines. A Linux block device driver communicates with the processes using TCP. The processes themselves communicate using TCP too. The processes can crash and recover at any time. A number of the processes is fixed before the system is run, and every process is given its own directory where it can store its internal files.</p>
<p>The smallest physical unit inside a block device is called a <strong>sector</strong>, and its size is specific to each device. A size of a block is always a multiple of the size of the sector. In the Atomic Disk Drive every sector is a separate atomic value called <strong>register</strong> (and thus it is said that the system supports a set of atomic values/registers). The sector has a size of 4096 bytes.</p>
<p>As follows from the above description, a complete Atomic Disk Drive consists of two parts: a Linux block device driver, and a user-space library implementing the distributed register. The Linux block device driver is provided in the package (see <a href="./linux_driver.html">instructions</a> on how to use it), and you can use it to test your solution. Your task is to implement in Rust the user-space part as the distributed system.</p>
<h3 id="distributed-register">Distributed register</h3>
<p>Your implementation of the distributed register shall be based on an algorithm named <strong>(N, N)-AtomicRegister</strong>.</p>
<h4 id="n-n-atomicregister">(N, N)-AtomicRegister</h4>
<p>There is a fixed number of instances of the AtomicRegister module, <code>N</code>, and all instances know about each other. Crashes of individual instances can happen. Every instance can initiate both read and write operations (thus the <code>(N, N)</code> in the name of the algorithm). It is assumed that the system is able to progress on operations as long as at least the majority of the instances are working correctly.</p>
<p>The core algorithm, based on the <em>Reliable and Secure Distributed Programming</em> by C. Cachin, R. Guerraoui, L. Rodrigues and modified to suit crash-recovery model, is as follows:</p>
<div class="sourceCode" id="cb1"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb1-1"><a href="#cb1-1"></a>Implements:</span>
<span id="cb1-2"><a href="#cb1-2"></a>    (N,N)-AtomicRegister instance nnar.</span>
<span id="cb1-3"><a href="#cb1-3"></a></span>
<span id="cb1-4"><a href="#cb1-4"></a>Uses:</span>
<span id="cb1-5"><a href="#cb1-5"></a>    StubbornBestEffortBroadcast, instance sbeb;</span>
<span id="cb1-6"><a href="#cb1-6"></a>    StubbornLinks, instance sl;</span>
<span id="cb1-7"><a href="#cb1-7"></a></span>
<span id="cb1-8"><a href="#cb1-8"></a>upon event &lt; nnar, Init &gt; do</span>
<span id="cb1-9"><a href="#cb1-9"></a>    (ts, wr, val) := (0, 0, _);</span>
<span id="cb1-10"><a href="#cb1-10"></a>    rid:= 0;</span>
<span id="cb1-11"><a href="#cb1-11"></a>    readlist := [ _ ] `of length` N;</span>
<span id="cb1-12"><a href="#cb1-12"></a>    acklist := [ _ ] `of length` N;</span>
<span id="cb1-13"><a href="#cb1-13"></a>    reading := FALSE;</span>
<span id="cb1-14"><a href="#cb1-14"></a>    writing := FALSE;</span>
<span id="cb1-15"><a href="#cb1-15"></a>    writeval := _;</span>
<span id="cb1-16"><a href="#cb1-16"></a>    readval := _;</span>
<span id="cb1-17"><a href="#cb1-17"></a>    write_phase := FALSE;</span>
<span id="cb1-18"><a href="#cb1-18"></a>    store(wr, ts, val, rid<font color="red"><s>, writing, writeval</s></font>);</span>
<span id="cb1-19"><a href="#cb1-19"></a></span>
<span id="cb1-20"><a href="#cb1-20"></a>upon event &lt; nnar, Recovery &gt; do</span>
<span id="cb1-21"><a href="#cb1-21"></a>    retrieve(wr, ts, val, rid<font color="red"><s>, writing, writeval</s></font>);</span>
<span id="cb1-22"><a href="#cb1-22"></a>    readlist := [ _ ] `of length` N;</span>
<span id="cb1-23"><a href="#cb1-23"></a>    acklist := [ _ ]  `of length` N;</span>
<span id="cb1-24"><a href="#cb1-24"></a>    reading := FALSE;</span>
<span id="cb1-25"><a href="#cb1-25"></a>    readval := _;</span>
<span id="cb1-26"><a href="#cb1-26"></a>    write_phase := FALSE;</span>
<span id="cb1-27"><a href="#cb1-27"></a>    <font color="red">writing := FALSE;<s>if writing = TRUE then</s></font></span>
<span id="cb1-28"><a href="#cb1-28"></a>    <font color="red">writeval := _;<s>    trigger &lt; sbeb, Broadcast | [READ_PROC, rid] &gt;;</s></font></span>
<span id="cb1-29"><a href="#cb1-29"></a></span>
<span id="cb1-30"><a href="#cb1-30"></a>upon event &lt; nnar, Read &gt; do</span>
<span id="cb1-31"><a href="#cb1-31"></a>    rid := rid + 1;</span>
<span id="cb1-32"><a href="#cb1-32"></a>    store(rid);</span>
<span id="cb1-33"><a href="#cb1-33"></a>    readlist := [ _ ] `of length` N;</span>
<span id="cb1-34"><a href="#cb1-34"></a>    acklist := [ _ ] `of length` N;</span>
<span id="cb1-35"><a href="#cb1-35"></a>    reading := TRUE;</span>
<span id="cb1-36"><a href="#cb1-36"></a>    trigger &lt; sbeb, Broadcast | [READ_PROC, rid] &gt;;</span>
<span id="cb1-37"><a href="#cb1-37"></a></span>
<span id="cb1-38"><a href="#cb1-38"></a>upon event &lt; sbeb, Deliver | p [READ_PROC, r] &gt; do</span>
<span id="cb1-39"><a href="#cb1-39"></a>    trigger &lt; sl, Send | p, [VALUE, r, ts, wr, val] &gt;;</span>
<span id="cb1-40"><a href="#cb1-40"></a></span>
<span id="cb1-41"><a href="#cb1-41"></a>upon event &lt;sl, Deliver | q, [VALUE, r, ts&#39;, wr&#39;, v&#39;] &gt; such that r == rid and !write_phase do</span>
<span id="cb1-42"><a href="#cb1-42"></a>    readlist[q] := (ts&#39;, wr&#39;, v&#39;);</span>
<span id="cb1-43"><a href="#cb1-43"></a>    if #(readlist) &gt; N / 2 and (reading or writing) then</span>
<span id="cb1-44"><a href="#cb1-44"></a>        <font color="red">readlist[self] := (ts, wr, val);</font></span>
<span id="cb1-45"><a href="#cb1-45"></a>        (maxts, rr, readval) := highest(readlist);</span>
<span id="cb1-46"><a href="#cb1-46"></a>        readlist := [ _ ] `of length` N;</span>
<span id="cb1-47"><a href="#cb1-47"></a>        acklist := [ _ ] `of length` N;</span>
<span id="cb1-48"><a href="#cb1-48"></a>        write_phase := TRUE;</span>
<span id="cb1-49"><a href="#cb1-49"></a>        if reading = TRUE then</span>
<span id="cb1-50"><a href="#cb1-50"></a>            trigger &lt; sbeb, Broadcast | [WRITE_PROC, rid, maxts, rr, readval] &gt;;</span>
<span id="cb1-51"><a href="#cb1-51"></a>        else</span>
<span id="cb1-52"><a href="#cb1-52"></a>            <font color="red">(ts, wr, val) := (maxts + 1, rank(self), writeval);</font></span>
<span id="cb1-53"><a href="#cb1-53"></a>            <font color="red">store(ts, wr, val);</font></span>
<span id="cb1-54"><a href="#cb1-54"></a>            trigger &lt; sbeb, Broadcast | [WRITE_PROC, rid, maxts + 1, rank(self), writeval] &gt;;</span>
<span id="cb1-55"><a href="#cb1-55"></a></span>
<span id="cb1-56"><a href="#cb1-56"></a>upon event &lt; nnar, Write | v &gt; do</span>
<span id="cb1-57"><a href="#cb1-57"></a>    rid := rid + 1;</span>
<span id="cb1-58"><a href="#cb1-58"></a>    writeval := v;</span>
<span id="cb1-59"><a href="#cb1-59"></a>    acklist := [ _ ] `of length` N;</span>
<span id="cb1-60"><a href="#cb1-60"></a>    readlist := [ _ ] `of length` N;</span>
<span id="cb1-61"><a href="#cb1-61"></a>    writing := TRUE;</span>
<span id="cb1-62"><a href="#cb1-62"></a>    store(rid<font color="red"><s>, writeval, writing</s></font>);</span>
<span id="cb1-63"><a href="#cb1-63"></a>    trigger &lt; sbeb, Broadcast | [READ_PROC, rid] &gt;;</span>
<span id="cb1-64"><a href="#cb1-64"></a></span>
<span id="cb1-65"><a href="#cb1-65"></a>upon event &lt; sbeb, Deliver | p, [WRITE_PROC, r, ts&#39;, wr&#39;, v&#39;] &gt; do</span>
<span id="cb1-66"><a href="#cb1-66"></a>    if (ts&#39;, wr&#39;) &gt; (ts, wr) then</span>
<span id="cb1-67"><a href="#cb1-67"></a>        (ts, wr, val) := (ts&#39;, wr&#39;, v&#39;);</span>
<span id="cb1-68"><a href="#cb1-68"></a>        store(ts, wr, val);</span>
<span id="cb1-69"><a href="#cb1-69"></a>    trigger &lt; sl, Send | p, [ACK, r] &gt;;</span>
<span id="cb1-70"><a href="#cb1-70"></a></span>
<span id="cb1-71"><a href="#cb1-71"></a>upon event &lt; sl, Deliver | q, [ACK, r] &gt; such that r == rid and write_phase do</span>
<span id="cb1-72"><a href="#cb1-72"></a>    acklist[q] := Ack;</span>
<span id="cb1-73"><a href="#cb1-73"></a>    if #(acklist) &gt; N / 2 and (reading or writing) then</span>
<span id="cb1-74"><a href="#cb1-74"></a>        acklist := [ _ ] `of length` N;</span>
<span id="cb1-75"><a href="#cb1-75"></a>        write_phase := FALSE;</span>
<span id="cb1-76"><a href="#cb1-76"></a>        if reading = TRUE then</span>
<span id="cb1-77"><a href="#cb1-77"></a>            reading := FALSE;</span>
<span id="cb1-78"><a href="#cb1-78"></a>            trigger &lt; nnar, ReadReturn | readval &gt;;</span>
<span id="cb1-79"><a href="#cb1-79"></a>        else</span>
<span id="cb1-80"><a href="#cb1-80"></a>            writing := FALSE;</span>
<span id="cb1-81"><a href="#cb1-81"></a>            <font color="red"><s>store(writing);</s></font></span>
<span id="cb1-82"><a href="#cb1-82"></a>            trigger &lt; nnar, WriteReturn &gt;;</span></code></pre></div>
<p>The <code>rank(*)</code> returns a rank of an instance, which is a static number assigned to an instance. The <code>highest(*)</code> returns the largest value ordered by <code>(timestamp, rank)</code>.</p>
<p>Your solution will not be receiving special <code>Recovery</code> or <code>Init</code> events. Each time it starts, it shall try to recover from its stable storage (during the initial run, the stable storage will be empty). Crashes are expected to happen at any point, your solution shall work despite them. The algorithm presented above is only a pseudocode, so we suggest understanding ideas behind it.</p>
<h4 id="linearization">Linearization</h4>
<p>Usually, components of a distributed system do not share a common clock. Atomic Disk Device does not have one too, and thus the events can happen at different rates and in various orders in every process. However, the atomic register enforces constraints between events on processes, and thereby it makes it possible to put all read and write operations on a single timeline, and to mark start and end of each operation. Every read returns the most recently written value. If an operation <code>o</code> happens before operation <code>o&#39;</code> when the system is processing messages, then <code>o</code> must appear before <code>o&#39;</code> on a such common timeline. This is called <em>linearization</em>.</p>
<p>To sum up, from a perspective of as single client (e.g., the Linux driver), there is a single sequence of read and write events. The client would not able to distinguish between the distributed register and some single device if it was performing the operations instead.</p>
<h4 id="performance">Performance</h4>
<p>The atomic register algorithm, as presented above, can only progress with one read or write operation at a time. However, this restriction applies only to a single value. Therefore, to improve performance of Atomic Disk Device, one can run multiple instances of the atomic register logic, each progressing on a different sector. It is expected that anything above a bare-minimum solution will provide such kind of concurrency and will be able to process many sectors at once.</p>
<h3 id="solution-specification">Solution specification</h3>
<p>Your solution shall take the form of a cargo library crate. Its main function is <code>run_register_process()</code> from the <code>solution/src/lib.rs</code> file, which shall run a new process of the distributed register. This function will be used by a simple wrapper—program to run your solution. The process will be passed all necessary information (e.g., addresses for TCP communication, directory for a stable storage, HMAC keys, etc.) via the <code>Configuration</code> struct (<code>solution/src/domain.rs</code>) provided as an argument to the function. Each processes of the distributed register will be run in a separate Linux processes.</p>
<p>The solution shall be asynchronous. It will be run using Tokio.</p>
<h4 id="tcp-communication">TCP communication</h4>
<p>Components of Atomic Disk Device shall communicate using TCP. The communication shall work as follows:</p>
<h5 id="client-to-process-communication">Client-to-process communication</h5>
<p>In the following description, we name a <em>client</em> any application communicating with the distributed register using TCP. The target client is the Linux device driver which is a part of Atomic Disk Drive. However, other clients will be also used when evaluating your solution (for instance, when unit testing).</p>
<p>Every process of the system can be contacted by a client. Clients connect using TCP, and send <code>READ</code> and <code>WRITE</code> commands. Your process must issue replies after a particular command is safely completed by the distributed register. Semantics of commands are mostly self-explanatory, so only their format is presented below. All numbers in the messages are always in the network byte order.</p>
<p><code>READ</code> and <code>WRITE</code> commands have the following format:</p>
<div class="sourceCode" id="cb2"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb2-1"><a href="#cb2-1"></a>0       7 8     15 16    23 24    31 32    39 40    47 48    55 56    64</span>
<span id="cb2-2"><a href="#cb2-2"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-3"><a href="#cb2-3"></a>|             Magic number          |       Padding            |   Msg  |</span>
<span id="cb2-4"><a href="#cb2-4"></a>| 0x61     0x74      0x64     0x64  |                          |   Type |</span>
<span id="cb2-5"><a href="#cb2-5"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-6"><a href="#cb2-6"></a>|                           Request number                              |</span>
<span id="cb2-7"><a href="#cb2-7"></a>|                                                                       |</span>
<span id="cb2-8"><a href="#cb2-8"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-9"><a href="#cb2-9"></a>|                           Sector index                                |</span>
<span id="cb2-10"><a href="#cb2-10"></a>|                                                                       |</span>
<span id="cb2-11"><a href="#cb2-11"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-12"><a href="#cb2-12"></a>|                           Command content ...</span>
<span id="cb2-13"><a href="#cb2-13"></a>|</span>
<span id="cb2-14"><a href="#cb2-14"></a>+--------+--------+--------...</span>
<span id="cb2-15"><a href="#cb2-15"></a>|                           HMAC tag                                    |</span>
<span id="cb2-16"><a href="#cb2-16"></a>|                                                                       |</span>
<span id="cb2-17"><a href="#cb2-17"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-18"><a href="#cb2-18"></a>|                           HMAC tag                                    |</span>
<span id="cb2-19"><a href="#cb2-19"></a>|                                                                       |</span>
<span id="cb2-20"><a href="#cb2-20"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-21"><a href="#cb2-21"></a>|                           HMAC tag                                    |</span>
<span id="cb2-22"><a href="#cb2-22"></a>|                                                                       |</span>
<span id="cb2-23"><a href="#cb2-23"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb2-24"><a href="#cb2-24"></a>|                           HMAC tag                                    |</span>
<span id="cb2-25"><a href="#cb2-25"></a>|                                                                       |</span>
<span id="cb2-26"><a href="#cb2-26"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span></code></pre></div>
<p><code>READ</code> operation type is <code>0x01</code>, <code>WRITE</code> operation type is <code>0x02</code>. Client can use the request number for internal identification of messages.</p>
<p><code>WRITE</code> has content of 4096 bytes to be written. <code>READ</code> has no content. The HMAC tag is a <code>hmac(sh256)</code> tag of the entire message (from the magic number to the end of the content).</p>
<p>After the system completes any of these two operations, it shall reply with a response of the following format:</p>
<div class="sourceCode" id="cb3"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb3-1"><a href="#cb3-1"></a>0       7 8     15 16    23 24    31 32    39 40    47 48    55 56    64</span>
<span id="cb3-2"><a href="#cb3-2"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb3-3"><a href="#cb3-3"></a>|             Magic number          |     Padding     | Status |  Msg   |</span>
<span id="cb3-4"><a href="#cb3-4"></a>| 0x61     0x74      0x64     0x64  |                 |  Code  |  Type  |</span>
<span id="cb3-5"><a href="#cb3-5"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb3-6"><a href="#cb3-6"></a>|                           Request number                              |</span>
<span id="cb3-7"><a href="#cb3-7"></a>|                                                                       |</span>
<span id="cb3-8"><a href="#cb3-8"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb3-9"><a href="#cb3-9"></a>|                           Response content ...</span>
<span id="cb3-10"><a href="#cb3-10"></a>|</span>
<span id="cb3-11"><a href="#cb3-11"></a>+--------+--------+--------...</span>
<span id="cb3-12"><a href="#cb3-12"></a>|                           HMAC tag                                    |</span>
<span id="cb3-13"><a href="#cb3-13"></a>|                                                                       |</span>
<span id="cb3-14"><a href="#cb3-14"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb3-15"><a href="#cb3-15"></a>|                           HMAC tag                                    |</span>
<span id="cb3-16"><a href="#cb3-16"></a>|                                                                       |</span>
<span id="cb3-17"><a href="#cb3-17"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb3-18"><a href="#cb3-18"></a>|                           HMAC tag                                    |</span>
<span id="cb3-19"><a href="#cb3-19"></a>|                                                                       |</span>
<span id="cb3-20"><a href="#cb3-20"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb3-21"><a href="#cb3-21"></a>|                           HMAC tag                                    |</span>
<span id="cb3-22"><a href="#cb3-22"></a>|                                                                       |</span>
<span id="cb3-23"><a href="#cb3-23"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span></code></pre></div>
<p>Again, the HMAC tag is <code>hmac(sha256)</code> of the entire response.</p>
<p>Possible status codes are listed in <code>StatusCode</code> (<code>solution/src/domain.rs</code>), and it is documented there when each code is expected. Status codes shall be consecutive numbers (starting with <code>0x00</code> as <code>Ok</code>), and be encoded as a single byte.</p>
<p>The response content for a successful <code>WRITE</code> is empty, for a successful <code>READ</code> it is 4096 bytes read from the system. If a command fails for any reason, the corresponding response shall have an empty content. The response message type is always <code>0x40</code> added to original message type: response type for <code>READ</code> is <code>0x41</code>, response type for <code>WRITE</code> is <code>0x42</code>.</p>
<p>Requests with an invalid HMAC tag shall be discarded with an appropriate status code returned. A HMAC key for client commands and responses is provided in <code>Configuration.hmac_client_key</code> (<code>solution/src/domain.rs</code>). When evaluating your solution, the key will be provided to the clients by a testing framework.</p>
<p>You can assume that a client will not send multiple commands with the same sector index at the same time. In other words, if a client sends a command regarding some sector, it will wait for the response before sending another command regarding this sector.</p>
<p>Operations submitted by clients can be executed in an arbitrary order. The system shall start sending a response when a command is completed by the register.</p>
<p>Since one atomic register can execute only one operation at a time (for a given sector), the operations shall be queued. We suggest using a TCP buffer itself as the queue.</p>
<h5 id="process-to-process-communication">Process-to-process communication</h5>
<p>All internal messages (i.e., messages sent between processes) shall have a common header as follows:</p>
<div class="sourceCode" id="cb4"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb4-1"><a href="#cb4-1"></a>0       7 8     15 16    23 24    31 32    39 40    47 48    55 56    64</span>
<span id="cb4-2"><a href="#cb4-2"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-3"><a href="#cb4-3"></a>|             Magic number          |       Padding   | Process|  Msg   |</span>
<span id="cb4-4"><a href="#cb4-4"></a>| 0x61     0x74      0x64     0x64  |                 |  Rank  |  Type  |</span>
<span id="cb4-5"><a href="#cb4-5"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-6"><a href="#cb4-6"></a>|                           UUID                                        |</span>
<span id="cb4-7"><a href="#cb4-7"></a>|                                                                       |</span>
<span id="cb4-8"><a href="#cb4-8"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-9"><a href="#cb4-9"></a>|                           UUID                                        |</span>
<span id="cb4-10"><a href="#cb4-10"></a>|                                                                       |</span>
<span id="cb4-11"><a href="#cb4-11"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-12"><a href="#cb4-12"></a>|                           Read identifier                             |</span>
<span id="cb4-13"><a href="#cb4-13"></a>|                        of register operation                          |</span>
<span id="cb4-14"><a href="#cb4-14"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-15"><a href="#cb4-15"></a>|                           Sector                                      |</span>
<span id="cb4-16"><a href="#cb4-16"></a>|                           index                                       |</span>
<span id="cb4-17"><a href="#cb4-17"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-18"><a href="#cb4-18"></a>|                           Message content ...</span>
<span id="cb4-19"><a href="#cb4-19"></a>|</span>
<span id="cb4-20"><a href="#cb4-20"></a>+--------+--------+--------...</span>
<span id="cb4-21"><a href="#cb4-21"></a>|                           HMAC tag                                    |</span>
<span id="cb4-22"><a href="#cb4-22"></a>|                                                                       |</span>
<span id="cb4-23"><a href="#cb4-23"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-24"><a href="#cb4-24"></a>|                           HMAC tag                                    |</span>
<span id="cb4-25"><a href="#cb4-25"></a>|                                                                       |</span>
<span id="cb4-26"><a href="#cb4-26"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-27"><a href="#cb4-27"></a>|                           HMAC tag                                    |</span>
<span id="cb4-28"><a href="#cb4-28"></a>|                                                                       |</span>
<span id="cb4-29"><a href="#cb4-29"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb4-30"><a href="#cb4-30"></a>|                           HMAC tag                                    |</span>
<span id="cb4-31"><a href="#cb4-31"></a>|                                                                       |</span>
<span id="cb4-32"><a href="#cb4-32"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span></code></pre></div>
<p>These internal messages shall be signed with different HMAC key then messages to/from clients. The key is provided in <code>Configuration.hmac_system_key</code> (<code>solution/src/domain.rs</code>). The process rank is the rank of a process which sends the command. The message types and their content are as follows:</p>
<ol type="1">
<li><p><code>READ_PROC</code></p>
<p>Type <code>0x03</code>, no content.</p></li>
<li><p><code>VALUE</code></p>
<p>Type <code>0x04</code>, content:</p>
<div class="sourceCode" id="cb5"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb5-1"><a href="#cb5-1"></a>0       7 8     15 16    23 24    31 32    39 40    47 48    55 56    64</span>
<span id="cb5-2"><a href="#cb5-2"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb5-3"><a href="#cb5-3"></a>|                           Timestamp                                   |</span>
<span id="cb5-4"><a href="#cb5-4"></a>|                                                                       |</span>
<span id="cb5-5"><a href="#cb5-5"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb5-6"><a href="#cb5-6"></a>|                 Padding                                      | Value  |</span>
<span id="cb5-7"><a href="#cb5-7"></a>|                                                              | wr     |</span>
<span id="cb5-8"><a href="#cb5-8"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb5-9"><a href="#cb5-9"></a>|                           Sector data ...</span>
<span id="cb5-10"><a href="#cb5-10"></a>|</span>
<span id="cb5-11"><a href="#cb5-11"></a>+--------+--------+--------...</span></code></pre></div>
<p>The <code>value wr</code> is a write rank of a process which delivered the last write. The sector data is 4096 bytes long.</p></li>
<li><p><code>WRITE_PROC</code></p>
<p>Type <code>0x05</code>, content:</p>
<div class="sourceCode" id="cb6"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb6-1"><a href="#cb6-1"></a>0       7 8     15 16    23 24    31 32    39 40    47 48    55 56    64</span>
<span id="cb6-2"><a href="#cb6-2"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb6-3"><a href="#cb6-3"></a>|                           Timestamp                                   |</span>
<span id="cb6-4"><a href="#cb6-4"></a>|                                                                       |</span>
<span id="cb6-5"><a href="#cb6-5"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb6-6"><a href="#cb6-6"></a>|                 Padding                                      | Value  |</span>
<span id="cb6-7"><a href="#cb6-7"></a>|                                                              | wr     |</span>
<span id="cb6-8"><a href="#cb6-8"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb6-9"><a href="#cb6-9"></a>|                           Sector data ...</span>
<span id="cb6-10"><a href="#cb6-10"></a>|</span>
<span id="cb6-11"><a href="#cb6-11"></a>+--------+--------+--------...</span></code></pre></div>
<p>The sector data contains 4096 bytes of data to be written. The <code>value wr</code> is a write rank of a process which delivered the last write.</p></li>
<li><p><code>ACK</code></p>
<p>Type <code>0x06</code>, no content.</p></li>
</ol>
<p>Moreover, you are allowed to use acknowledgment responses to the internal messages. They shall contain the UUID of a message that is being acknowledged, and they shall be signed with the <code>Configuration.hmac_system_key</code> HMAC key too. The message format is:</p>
<div class="sourceCode" id="cb7"><pre class="sourceCode numberSource numberLines"><code class="sourceCode"><span id="cb7-1"><a href="#cb7-1"></a>0       7 8     15 16    23 24    31 32    39 40    47 48    55 56    64</span>
<span id="cb7-2"><a href="#cb7-2"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-3"><a href="#cb7-3"></a>|             Magic number          |     Padding     |Process |  Msg   |</span>
<span id="cb7-4"><a href="#cb7-4"></a>| 0x61     0x74      0x64     0x64  |                 |  Rank  |  Type  |</span>
<span id="cb7-5"><a href="#cb7-5"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-6"><a href="#cb7-6"></a>|                           UUID                                        |</span>
<span id="cb7-7"><a href="#cb7-7"></a>|                                                                       |</span>
<span id="cb7-8"><a href="#cb7-8"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-9"><a href="#cb7-9"></a>|                           UUID                                        |</span>
<span id="cb7-10"><a href="#cb7-10"></a>|                                                                       |</span>
<span id="cb7-11"><a href="#cb7-11"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-12"><a href="#cb7-12"></a>|                           HMAC tag                                    |</span>
<span id="cb7-13"><a href="#cb7-13"></a>|                                                                       |</span>
<span id="cb7-14"><a href="#cb7-14"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-15"><a href="#cb7-15"></a>|                           HMAC tag                                    |</span>
<span id="cb7-16"><a href="#cb7-16"></a>|                                                                       |</span>
<span id="cb7-17"><a href="#cb7-17"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-18"><a href="#cb7-18"></a>|                           HMAC tag                                    |</span>
<span id="cb7-19"><a href="#cb7-19"></a>|                                                                       |</span>
<span id="cb7-20"><a href="#cb7-20"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span>
<span id="cb7-21"><a href="#cb7-21"></a>|                           HMAC tag                                    |</span>
<span id="cb7-22"><a href="#cb7-22"></a>|                                                                       |</span>
<span id="cb7-23"><a href="#cb7-23"></a>+--------+--------+--------+--------+--------+--------+--------+--------+</span></code></pre></div>
<p>The process rank is the rank of a process which sends the response (i.e., acknowledges the message). The message type is always a <code>0x40</code> added to the type of the original message. For example, it is <code>0x46</code> for <code>ACK</code> (as <code>0x06 + 0x40 = 0x46</code>).</p>
<h5 id="handling-incorrect-messages">Handling incorrect messages</h5>
<p>When a message which does not comply with the presented above formats is received, it shall be handled as follows:</p>
<ul>
<li>The solution shall slide over bytes in the stream until it detects a valid magic number. This marks a beginning of a message.</li>
<li>If a message type is invalid, the solution shall discard the magic number and the following 4 bytes (8 bytes in total).</li>
<li>In case of every other error, the solution shall consume the same number of bytes as if a message of this type was processed successfully.</li>
</ul>
<h5 id="other-requirements">Other requirements</h5>
<p>We expect that within 300 milliseconds after calling <code>run_register_process()</code> a TCP socket will be bound. We suggest the function binds to the appropriate socket before executing other actions.</p>
<p>Your internal TCP client is not allowed to lose any messages, even when a target process crashes and the TCP connection gets broken.</p>
<p>You are allowed to use custom messages, for instance, for debugging purposes. Their message types shall be numbers equal to or greater than <code>0x80</code>.</p>
<p>It is suggested that messages sent by a process to itself should skip serialization, deserialization, HMAC preparation and validation phases to improve the performance.</p>
<h4 id="internal-interfaces">Internal interfaces</h4>
<p>As mentioned above, the main function of the assignment is <code>run_register_process()</code>, and you will receive the maximum number of point if you implement it perfectly. However, to efficiently and fairly grade solutions which are overall correct but have minor errors, your solution will be also evaluated with unit tests. To this end, the template splits the implementation of the distributed register into multiple parts and defines their interfaces. Most of the interfaces are asynchronous since running the register will result in multiple IO tasks, and cooperative multitasking seems to be notably profitable in such application.</p>
<p>In the package we provide a diagram (<code>atdd.svg</code>) presenting how Atomic Disk Device might be implemented. Every process of the distributed regiester is wrapped in a Linux process. Tokio is used as the executor system. The Linux block device driver sends commands to processes over TCP. Processes communicate with one another using internal messages, and then complete the commands and return responses over TCP back to the driver. Every process of the distributed register has multiple copies of the atomic register code, to support concurrent writes/reads on distinct sectors. The copies share a component which handles communication (<code>RegisterClient</code>), and a component which handles storing sectors data.</p>
<p>Your solution shall implement the following interfaces (they are defined in <code>solution/src/lib.rs</code>):</p>
<h5 id="atomicregister">AtomicRegister</h5>
<p><code>AtomicRegister</code> provides functionality required of an instance of the atomic register algorithm. When it is being created by calling <code>build_atomic_register()</code> (<code>solution/src/lib.rs</code>), it shall attempt to recover from its stable storage. All its methods shall follow the atomic register algorithm presented above. When implementing <code>AtomicRegister</code>, you can assume that <code>RegisterClient</code> passed to the function implements <code>StubbornLink</code> required by the algorithm.</p>
<p>Every sector is logically a separate atomic register. However, an <code>AtomicRegister</code> object shall <strong>not</strong> correspond to a single sector. At any time any <code>AtomicRegister</code> object shall be progressing at most one client command, but in general it shall respond to <code>READ_PROC</code> and <code>WRITE_PROC</code> messages regarding other sectors, as their handlers only refer to the state of sectors storage. Of course, you shall be careful not to create a race condition when selecting an <code>AtomicRegister</code> object for serving a command. To avoid collisions between messages from different reads and writes you can use the <code>UUID</code> field. We do not specify how to use it and, in particular, it does not have to be random or unique for every message.</p>
<h5 id="stablestorage">StableStorage</h5>
<p>It might be convenient to separate storage of the sectors data from storage of data (metadata) required to run the atomic register algorithm. The latter, <code>StableStorage</code>, is a key-value stable storage. It is expected that you reuse your implementation of stable storage from the recent Small Assignment.</p>
<h5 id="sectorsmanager">SectorsManager</h5>
<p><code>SectorsManager</code> facilitates storing sectors data in the filesystem directory. Sector data shall be stored together with with neccessary basic information, such as the logical timestamp and the write rank (see the pseudocode of the atomic register algorithm).</p>
<p>Sectors are numbered from <code>0</code> inclusive to <code>Configuration.public.max_sector</code> (<code>solution/src/domain.rs</code>) exclusive. You can assume that <code>Configuration.public.max_sector</code> will not exceed <code>2^21</code>.</p>
<p>If a sector was never written, we assume that both the logical timestamp and the write rank are <code>0</code>, and that it contains 4096 zero bytes.</p>
<p>No particular storage scheme is required, it shall just provide atomic operations. No caching is necessary.</p>
<p>The <code>build_sectors_manager()</code> function (<code>solution/src/lib.rs</code>) shall create an instance of <code>SectorManager</code> for, among others, unit testing. You can assume that the unit tests will not perform concurrent operations on the same sector, even though the trait is marked as <em>Sync</em>.</p>
<p>The separation of <code>StableStorage</code> and <code>SectorsManager</code> is introduced due to efficiency reasons, as the main storage for sectors data is not expected to be a general purpose storage. Both storage types running within a single process shall share a directory provided in <code>Configuration.public.storage_dir</code> (<code>solution/src/domain.rs</code>). However, you are allowed to create there as many subdirectories as you wish, for any purpose.</p>
<h5 id="registerclient">RegisterClient</h5>
<p><code>RegisterClient</code> manages TCP communication between processes of the distributed register. An instance is passed to instances of <code>AtomicRegister</code> to allow them communicating with each other.</p>
<p>This trait is introduced mainly of the purpose of unit testing.</p>
<p>When sending messages to itself, it is suggested to transfer the messages in some more direct manner than TCP, to increase the performance.</p>
<h5 id="serialization-and-deserialization">Serialization and deserialization</h5>
<p>Concerning serialization, your solution shall provide two methods: <code>deserialize_register_command()</code> and <code>serialize_register_command()</code>. They convert bytes to a <code>RegisterCommand</code> object and in the other direction, respectively. They shall implement the message formats as described above (see the description of TCP communication).</p>
<p>Serialization shall complete successfully when there are no errors when writing to the provided reference that implements <code>AsyncWrite</code>. If errors occur, the serializing function shall return them.</p>
<p>Deserialization shall return a pair <code>(message, hmac_valid)</code> when a valid magic number and a valid message type is encountered. Incorrect messages shall be handled as described above (see the description of TCP communication). An error shall be returned if an error occurs when reading from the provided <code>AsyncRead</code> reference.</p>
<h4 id="technical-requirements">Technical requirements</h4>
<p>Your solution will be tested with the stable Rust version <code>rustc 1.56.1 (59eed8a2a 2021-11-01)</code>.</p>
<h5 id="interface">Interface</h5>
<p>You must not modify the public interface of the library. You can, however, implement more derivable traits for public types via the <code>derive</code> attribute.</p>
<h5 id="dependencies">Dependencies</h5>
<p>You can use only crates listed in <code>solution/Cargo.toml</code> for main dependencies, and anything you want for <code>[dev-dependencies]</code> section, as this section will be ignored when evaluating your solution. You are not allowed to use <code>[build-dependencies]</code>, but you can specify any number of binaries in your <code>Cargo.toml</code> (these will also be ignored during evaluation). If you need any other crate, ask on Moodle for a permission. If a permit is granted, every student is allowed to use it.</p>
<p>Using asynchronous libraries makes it easy to scale your solution to the number of available cores, and wait for completions of hundreds of concurrent IO tasks. This is necessary to reach an acceptable performance.</p>
<h5 id="storage">Storage</h5>
<p>Because crashes are to be expected, the <code>Configuration</code> struct specifies a directory for an exclusive use by a process. You can use it, for instance, to implement the stable storage on top of it. You are allowed to create subdirectories within the directory. You are not allowed to touch any other directory.</p>
<p>There is a limit on the number of open file descriptors, 1024. We suggest utilizing it for a maximum concurrency. You can assume that there will not be more than 16 client connections.</p>
<p>You can assume that a local filesystem stores data in blocks of 4096 bytes, the same size as the sectors.</p>
<p>Your solution is allowed to use at most 10% more filesystem space than the size of sectors which are stored. That is, if <code>n</code> distinct sectors are stored, it is expected that the total directory size does not exceed <code>1.1 * n * 4096</code> bytes (assuming <code>n &gt;= 1000</code>).</p>
<h5 id="logging">Logging</h5>
<p>You are allowed to use logging as you wish, as long as your solution does not produce a huge volume of messages at levels <code>&gt;= INFO</code> when the system is operating correctly.</p>
<h5 id="performance-1">Performance</h5>
<p>It is expected that a satisfactory solution implements a distributed register which, when run with a single process on a computer from lab 2041 (codename red), processes on average at least 25 sectors per second per logical core.</p>
<h3 id="assignment-specification">Assignment specification</h3>
<p>You are given a subset of official tests (see <code>public-tests/</code> in the package). Their intention is to make sure that the public interface of your solution is correct, and to evaluate basic functionality.</p>
<h4 id="submitting-solution">Submitting solution</h4>
<p>Your solution must be submitted as a single <code>.zip</code> file with its name being your login at students (e.g., <code>ab123456.zip</code>). After unpacking the archive, a directory path named <code>ab123456/solution/</code> must be created. In the <code>solution</code> subdirectory there must be a Rust library crate that implements the assignment. Project <code>public-tests</code> must be able to be built and tested cleanly when placed next to the <code>solution</code> directory.</p>
<h4 id="grading">Grading</h4>
<p>Your solution will be graded based on results of automatic tests and code inspection. The number of available and required points is specified in the <a href="../../">Passing Rules</a> described at the main website of the course. If your solution passes the public tests, you will receive at least the required number of points. Solutions which will not actually implement a distributed system (e.g., they will keep all data in RAM only, execute commands on a single node, etc.) will be rejected.</p>
<hr />
<p>Authors: F. Plata, K. Iwanicki, M. Banaszek, W. Ciszewski.</p>
</section>

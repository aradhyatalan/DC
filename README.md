Inter Process Communication:

//Client.java
import java.io.*;
import java.util.*;
import java.net.*;
public class Client
{
public static void main(String args[])throws Exception
{
String send="",r="";
Socket MyClient = new Socket("192.168.29.4",25);
DataInputStream din=new DataInputStream(MyClient.getInputStream());
DataOutputStream dout = new DataOutputStream(MyClient.getOutputStream());
Scanner sc = new Scanner(System.in);
while(!send.equals("stop")){
System.out.print("Send: ");
send = sc.nextLine();
dout.writeUTF(send);
}
dout.flush();
r=din.readUTF();
System.out.println("Reply: "+ r);
dout.close();
din.close();
MyClient.close();
}
}
//Server.java
import java.util.*;
import java.io.*;
import java.net.*;
public class Server {
public static void main(String args[]) throws Exception{
ServerSocket MyServer = new ServerSocket(25);
Socket ss = MyServer.accept();
DataInputStream din =new DataInputStream(ss.getInputStream());
DataOutputStream dout=new DataOutputStream(ss.getOutputStream());
BufferedReader br=new BufferedReader(new InputStreamReader(System.in));
Server server = new Server();
String str="",str2="";
int sum = 0;
while(!str.equals("stop")){
str=din.readUTF();
if(str.equals("stop"))
break;
sum = sum + Integer.parseInt(str);
}
dout.writeUTF(Integer.toString(sum));
dout.flush();
din.close();
ss.close();
MyServer.close();
}
}



Group Communication:

//Client.java
import java.io.IOException;
import java.io.PrintWriter;
import java.net.Socket;
import java.util.Scanner;
public class GroupChatClient {
public static void main(String[] args) throws IOException {
Scanner scanner = new Scanner(System.in);
System.out.print("Enter your name: ");
String clientName = scanner.nextLine();
Socket socket = new Socket("localhost", 5000);
try {
PrintWriter writer = new PrintWriter(socket.getOutputStream(),
true);
writer.println(clientName);
Thread readerThread = new Thread(new ClientReader(socket));
readerThread.start();
while (true) {
String message = scanner.nextLine();
writer.println(message);
if (message.equals("exit")) {
break;
}
}
} finally {
socket.close();
}
}
private static class ClientReader implements Runnable {
private Socket socket;
public ClientReader(Socket socket) {
this.socket = socket;
}
@Override
public void run() {
try {
Scanner scanner = new Scanner(socket.getInputStream());
while (scanner.hasNextLine()) {
String message = scanner.nextLine();
System.out.println(message);
}
} catch (IOException e) {
e.printStackTrace();
} finally {
System.out.println("Disconnected from the server.");
}
}
}
}
//Server.java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.HashSet;
import java.util.Scanner;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;
public class GroupChatServer {
private static Set<PrintWriter> writers = new CopyOnWriteArraySet<>();
private static Set<String> connectedClients = new HashSet<>();
public static void main(String[] args) throws IOException {
ServerSocket serverSocket = new ServerSocket(5000);
System.out.println("Group Chat Server is running...");
// Start a thread to accept messages from the server console
Thread serverMessageThread = new Thread(new
ServerMessageHandler());
serverMessageThread.start();
try {
while (true) {
Socket clientSocket = serverSocket.accept();
System.out.println("New client connected: " +
clientSocket);
Scanner clientScanner = new
Scanner(clientSocket.getInputStream());
String clientName = clientScanner.nextLine();
synchronized (writers) {
writers.add(new
PrintWriter(clientSocket.getOutputStream(), true));
connectedClients.add(clientName);
}
broadcastFromServer(clientName + " has joined the chat.");
broadcastFromServer("new member has joined !!");
Thread clientHandler = new Thread(new
ClientHandler(clientSocket, clientName));
clientHandler.start();
}
} finally {
serverSocket.close();
}
}
private static class ClientHandler implements Runnable {
private Socket clientSocket;
private String clientName;
public ClientHandler(Socket clientSocket, String clientName) {
this.clientSocket = clientSocket;
this.clientName = clientName;
}
@Override
public void run() {
try {
Scanner scanner = new
Scanner(clientSocket.getInputStream());
while (scanner.hasNextLine()) {
String message = scanner.nextLine();
if (message.equals("exit")) {
break;
} else if (message.startsWith("/whisper")) {
handleWhisper(message);
} else {
broadcast(clientName + ": " + message);
}
}
} catch (IOException e) {
e.printStackTrace();
} finally {
try {
clientSocket.close();
} catch (IOException e) {
e.printStackTrace();
}
synchronized (writers) {
writers.removeIf(writer ->
writer.equals(clientSocket));
connectedClients.remove(clientName);
}
broadcastFromServer(clientName + " has left the chat.");
}
}
private void broadcast(String message) {
synchronized (writers) {
for (PrintWriter writer : writers) {
writer.println(message);
}
}
}
private void handleWhisper(String message) {
String[] parts = message.split(" ", 3);
if (parts.length == 3) {
String recipient = parts[1].trim();
String whisperMessage = parts[2].trim();
synchronized (writers) {
if (connectedClients.contains(recipient)) {
for (PrintWriter writer : writers) {
if (writer.toString().equals(recipient)) {
writer.println("(whisper from " +
clientName + "): " + whisperMessage);
break;
}
}
} else {
writers.stream()
.filter(writer ->
writer.toString().equals(clientName))
.findFirst()
.ifPresent(writer ->
writer.println("Recipient '" + recipient + "' is not connected."));
}
}
} else {
writers.stream()
.filter(writer ->
writer.toString().equals(clientName))
.findFirst()
.ifPresent(writer -> writer.println("Invalid whisper command. Usage: /whisper recipient
message"));
}
}
}
private static class ServerMessageHandler implements Runnable {
@Override
public void run() {
try {
BufferedReader consoleReader = new BufferedReader(new
InputStreamReader(System.in));
while (true) {
String serverMessage = consoleReader.readLine();
broadcastFromServer("[host]: " + serverMessage);
}
} catch (IOException e) {
e.printStackTrace();
}
}
}
public static void broadcastFromServer(String message) {
synchronized (writers) {
for (PrintWriter writer : writers) {
writer.println(message);
}
}
}
}



Clock Synchronization – Lamport’s Logical Clock

import java.util.Scanner;
class LamportClock {
private int time;
public LamportClock(int initialTime) {
this.time = initialTime;
}
public synchronized int getTime() {
return time;
}
public synchronized void tick() {
time++;
}
public synchronized void sendAction() {
time++;
}
public synchronized void receiveAction(int sentTime) {
time = Math.max(time, sentTime) + 1;
}
}
class Process implements Runnable {
private int processId;
private LamportClock clock;
private Scanner scanner;
public Process(int processId, LamportClock clock) {
this.processId = processId;
this.clock = clock;
this.scanner = new Scanner(System.in);
}
@Override
public void run() {
System.out.println("Process " + processId + " initial time: " + clock.getTime());
while (true) {
System.out.println("Process " + processId + ": Enter 'send' to send packet or 'exit' to quit:");
String input = scanner.nextLine();
if (input.equals("exit")) {
break;
} else if (input.equals("send")) {
sendMessage();
} else {
System.out.println("Invalid input. Please enter 'send' or 'exit'.");
}
}
}
private synchronized void sendMessage() {
int sentTime = clock.getTime();
clock.sendAction();
System.out.println("Process " + processId + " sent a packet at time " + sentTime);
// Simulating packet transmission time
try {
Thread.sleep(1000); // Sleep for 1 second
} catch (InterruptedException e) {
e.printStackTrace();
}
// Receive the packet
int receivedTime = clock.getTime();
clock.receiveAction(sentTime);
System.out.println("Process " + processId + " received a packet at time " + receivedTime);
}
}
public class LamportClockSimulation {
public static void main(String[] args) {
LamportClock clock1 = new LamportClock(0); // Initial time for process 1
LamportClock clock2 = new LamportClock(5); // Initial time for process 2
Thread process1 = new Thread(new Process(1, clock1));
Thread process2 = new Thread(new Process(2, clock2));
process1.start();
process2.start();
}
}



Simulate election algorithm – Bully algorithm

import java.util.Scanner;
class Pro {
int id;
boolean act;
Pro(int id) {
this.id = id;
this.act = true;
}
}
public class Bully {
int TotalProcess;
Pro[] process;
public static void main(String[] args) {
Bully object = new Bully();
Scanner scanner = new Scanner(System.in);
System.out.print("Total number of processes: ");
int numProcesses = scanner.nextInt();
object.initialize(numProcesses);
object.Election();
scanner.close();
}
public void initialize(int j) {
System.out.println("No of processes " + j);
this.TotalProcess = j;
this.process = new Pro[this.TotalProcess];
for (int i = 0; i < this.TotalProcess; i++) {
this.process[i] = new Pro(i);
}
}
public void Election() {
Scanner scanner = new Scanner(System.in);
System.out.print("Enter the process number to fail: ");
int process_to_fail = scanner.nextInt();
if (process_to_fail < 0 || process_to_fail >= this.TotalProcess) {
System.out.println("Invalid process number. Please enter a valid process number.");
return;
}
System.out.println("Process no " + this.process[process_to_fail].id + " fails");
this.process[process_to_fail].act = false;
System.out.print("Enter the process initiating the election: ");
int initializedProcess = scanner.nextInt();
System.out.println("Election Initiated by " + initializedProcess);
int initBy = initializedProcess;
int i = -1;
for(i = initBy; i < TotalProcess - 1; i++){
if(this.process[i].act){
System.out.println("Process " + i + " passes Election(" + i + ")" + " to " +(i + 1));
}else{
System.out.println("Process " + (i-1) + " passes Election(" + (i - 1) + ")" + " to " +(i + 1));
}
}
int coordinatorID = -1;
if(this.process[i].act) {
coordinatorID = i;
}else{
coordinatorID = i - 1;
}
System.out.println("Process " + coordinatorID + " is the process with max ID");
for(int j = 0; j< coordinatorID; j++){
if(this.process[j].act)
System.out.println("Process " + coordinatorID + " passes Coordinator(" + coordinatorID + ")
message to process " + this.process[j].id);
}
scanner.close();
}
}



Simulate Mutual Exclusion – Raymond’s Tree Based Algorithm

import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.Scanner;
public class RaymondTree {
static List<Node> list = new ArrayList<>();
public static void main(String[] args) {
Scanner scanner = new Scanner(System.in);
int n = 8;
Node n1 = new Node(1, 1);
Node n2 = new Node(2, 1);
Node n3 = new Node(3, 1);
Node n4 = new Node(4, 2);
Node n5 = new Node(5, 2);
Node n6 = new Node(6, 3);
Node n7 = new Node(7, 3);
Node n8 = new Node(8, 3);
list.add(n1);
list.add(n2);
list.add(n3);
list.add(n4);
list.add(n5);
list.add(n6);
list.add(n7);
list.add(n8);
while (true) {
System.out.println("Enter the ID of the node requesting access to the critical section (1-8), or
'exit' to quit:");
String input = scanner.nextLine();
if (input.equalsIgnoreCase("exit")) {
break;
}
int nodeID;
try {
nodeID = Integer.parseInt(input);
if (nodeID < 1 || nodeID > 8) {
System.out.println("Invalid node ID. Please enter a number between 1 and 8.");
continue;
}
} catch (NumberFormatException e) {
System.out.println("Invalid input. Please enter a valid node ID or 'exit'.");
continue;
}
List<Integer> req = new LinkedList<>();
req.add(nodeID);
System.out.println("Initially:");
printFunction();
for (int a : req) {
System.out.println("--------- Process " + a + " ----------- ");
request(a, a);
}
}
scanner.close();
}
private static void request(int holder, int req) {
Node n = list.get(holder - 1);
n.q.add(req);
System.out.println("Queue of " + n.id + ": " + n.q);
if (n.holder != n.id)
request(n.holder, n.id);
else
giveToken(n.id);
}
private static void giveToken(int nid) {
Node n = list.get(nid - 1);
int next = n.q.remove();
n.holder = next;
if (next != nid)
giveToken(next);
else
printFunction();
}
private static void printFunction() {
for (Node n : list) {
System.out.println("id: " + n.id + " Holder: " + n.holder);
}
}
}
class Node {
int id;
int holder;
Queue<Integer> q;
Node(int id, int holder) {
this.id = id;
this.holder = holder;
this.q = new LinkedList<>();
}}



Deadlock Management – Banker’s algorithm

import java.util.Scanner;
public class BankersAlgorithm {
private int[][] allocation;
private int[][] max;
private int[] available;
private int[][] need;
private int numProcesses;
private int numResources;
public BankersAlgorithm(int numProcesses, int numResources) {
this.numProcesses = numProcesses;
this.numResources = numResources;
allocation = new int[numProcesses][numResources];
max = new int[numProcesses][numResources];
available = new int[numResources];
need = new int[numProcesses][numResources];
}
public void getInput() {
Scanner scanner = new Scanner(System.in);
// Input allocation matrix
System.out.println("Enter Allocation Matrix:");
for (int i = 0; i < numProcesses; i++) {
for (int j = 0; j < numResources; j++) {
allocation[i][j] = scanner.nextInt();
}
}
// Input max matrix
System.out.println("Enter Max Matrix:");
for (int i = 0; i < numProcesses; i++) {
for (int j = 0; j < numResources; j++) {
max[i][j] = scanner.nextInt();
need[i][j] = max[i][j] - allocation[i][j];
}
}
// Input available resources
System.out.println("Enter Available Resources:");
for (int i = 0; i < numResources; i++) {
available[i] = scanner.nextInt();
}
}
public boolean isSafe(int process) {
int[] tempAvailable = available.clone();
int[][] tempAllocation = new int[numProcesses][numResources];
int[][] tempNeed = new int[numProcesses][numResources];
// Check if the system is still in safe state after resource allocation
boolean[] finished = new boolean[numProcesses];
for (int i = 0; i < numProcesses; i++) {
finished[i] = false;
}
// Try to allocate the resources for the current process
for (int i = 0; i < numResources; i++) {
tempAvailable[i] -= need[process][i];
tempAllocation[process][i] += need[process][i];
tempNeed[process][i] -= need[process][i];
}
int count = 0;
while (count < numProcesses) {
boolean found = false;
for (int i = 0; i < numProcesses; i++) {
if (!finished[i]) {
boolean safe = true;
for (int j = 0; j < numResources; j++) {
if (tempNeed[i][j] > tempAvailable[j]) {
safe = false;
break;
}
}
if (safe) {
finished[i] = true;
count++;
found = true;
for (int j = 0; j < numResources; j++) {
tempAvailable[j] += tempAllocation[i][j];
}
}
}
}
if (!found) {
break;
}
}
return count == numProcesses;
}
public void grantResources() {
for (int i = 0; i < numProcesses; i++) {
if (isSafe(i)) {
for (int j = 0; j < numResources; j++) {
available[j] += allocation[i][j];
allocation[i][j] = 0;
need[i][j] = 0;
}
System.out.println("Resources granted for process " + (i + 1) + " successfully.");
System.out.println("Available resources after process " + (i + 1) + ":");
for (int k = 0; k < numResources; k++) {
System.out.print(available[k] + " ");
}
System.out.println();
} else {
System.out.println("Unsafe state, resources cannot be granted for process " + (i + 1) + ".");
}
}
}
public static void main(String[] args) {
Scanner scanner = new Scanner(System.in);
// Input number of processes and resources
System.out.println("Enter number of processes:");
int numProcesses = scanner.nextInt();
System.out.println("Enter number of resources:");
int numResources = scanner.nextInt();
BankersAlgorithm bankers = new BankersAlgorithm(numProcesses,
numResources);
bankers.getInput();
bankers.grantResources();
}
}



Load Balancing Algorithm

import java.util.Scanner;
class LoadBalancer{
static void printLoad(int servers,int Processes){
int each = Processes/servers;
int extra = Processes%servers;
int total = 0;
for(int i = 0; i < servers; i++){
if(extra-->0) total = each+1;
else total = each;
System.out.println("Server "+(char)('A'+i)+" has "+total+" Processes");
}
}
public static void main(String[] args){
Scanner sc = new Scanner(System.in);
System.out.print("Enter the number of servers and Processes: ");
int servers = sc.nextInt();
int Processes = sc.nextInt();
while(true){ printLoad(servers, Processes);
System.out.print("1.Add Servers 2.Remove Servers 3.Add Processes 4.Remove Processes
5.Exit: ");
switch(sc.nextInt()){
case 1:
System.out.print("How many more servers?: ");
servers+=sc.nextInt();
break;
case 2:
System.out.print("How many servers to remove?: ");
servers-=sc.nextInt();
break;
case 3:
System.out.print("How many more Processes?: ");
Processes+=sc.nextInt();
break;
case 4:
System.out.print("How many Processes to remove?: ");
Processes-=sc.nextInt();
break;
case 5: return;
}
}
}
}





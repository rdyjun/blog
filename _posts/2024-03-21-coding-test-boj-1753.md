---
title: "[백준] 1753번 최단경로 #JAVA"
excerpt: "다익스트라 알고리즘을 활용한 단방향 최단 경로 찾기 문제"

categories:
  - 코딩 테스트
tags:
  - [java, dijkstra, algorithm, 자바, 다익스트라, 알고리즘, priority queue, queue, 우선순위 큐]

permalink: /categories/코딩-테스트/boj-1753

toc: true
toc_sticky: true

date: 2024-03-21
last_modified_at: 2024-03-21
---

# 문제
https://www.acmicpc.net/problem/1753

# 풀이
최단 경로 탐색 문제기 때문에 다익스트라 방식으로 풀었다. 

처음엔 int[][] 배열을 사용해서 graph를 작성했다가, 시간초과가 떠서 HashMap을 사용하고, 좀 더 줄일 수 있을 것 같아 검색해보니, List를 사용하는 것이 더 효율적일 것 같아 결과적으로 이중 List를 사용했다.

## 흐름
값들을 모두 입력 받아, 각 노드의 거리를 담은 `graph`와 시작점으로 부터 다른 노드까지의 거리를 담는 `distance`를 초기화 해줬다.
`graph`는 `List<List<Node>>`로 해주었기 때문에 따로 `Integer.MAX_VALUE` 설정은 해주지 않았고, `distance`만 `Integer.MAX_VALUE` 설정을 해주었다.
그 다음 다익스트라 알고리즘을 통해 우선 순위 큐에서 비용이 가장 적은 노드를 꺼내어 해당 노드가 갈 수 있는 노드의 비용을 `graph list`에서 꺼내, `distance`에 기록된 비용과 비교해 더 작다면 업데이트 해주었다.



```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.PriorityQueue;
import java.util.Queue;
import java.util.StringTokenizer;

public class Main {

    private static int verticesCount;

    private static int edgeCount;

    private static int startVertices;

    private static List<List<Node>> graph;

    private static int[] distance;

    private static Queue<Node> priorityQueue;

    public static void main(String[] args) {
        IO.init();
        findMinCostOfNode();
        IO.close();
    }

    private static void findMinCostOfNode() {
        StringTokenizer input = new StringTokenizer(IO.read(), " ");
        verticesCount = Integer.parseInt(input.nextToken());
        edgeCount = Integer.parseInt(input.nextToken());
        startVertices = Integer.parseInt(IO.read());

        initializeGraph();
        initializeDistance();

        dijkstra();
        printAnswer();
    }

    private static void printAnswer() {
        for (int index = 0; index < verticesCount; index++) {
            if (distance[index] == Integer.MAX_VALUE) {
                IO.write("INF\n");
                continue;
            }

            IO.write(distance[index] + "\n");
        }
    }

    private static void dijkstra() {
        priorityQueue = new PriorityQueue<>();
        priorityQueue.offer(new Node(startVertices - 1, 0));

        while (!priorityQueue.isEmpty()) {
            Node now = priorityQueue.poll();

            updateNodeCost(now);
        }
    }

    private static void updateNodeCost(Node now) {
        List<Node> nextNodes = graph.get(now.getNode());
        int weight = now.getWeight();

		for (Node node : nextNodes) {
            int nodeNum = node.getNode();
            int sum = node.getWeight() + weight;

			if (sum < distance[nodeNum]) {
                priorityQueue.add(new Node(nodeNum, sum));
                distance[nodeNum] = sum;
			}
		}
    }

    private static void initializeDistance() {
        distance = new int[verticesCount];
        Arrays.fill(distance, Integer.MAX_VALUE);
        distance[startVertices - 1] = 0;
    }

    private static void initializeGraph() {
        graph = new ArrayList<>();
        for (int i = 0; i < verticesCount; i++) {
            graph.add(new ArrayList<>());
        }

        while (edgeCount-- > 0) {
            StringTokenizer input = new StringTokenizer(IO.read(), " ");

            int a = Integer.parseInt(input.nextToken()) - 1;
            int b = Integer.parseInt(input.nextToken()) - 1;
            int w = Integer.parseInt(input.nextToken());

            List<Node> arr = graph.get(a);
            arr.add(new Node(b, w));
            graph.set(a, arr);
        }
    }

}

class Node implements Comparable<Node> {

    private final Integer node;

    private final Integer weight;

    public Node(int node, int cost) {
        this.node = node;
        this.weight = cost;
    }

    @Override
    public int compareTo(Node other) {
        return Integer.compare(this.weight, other.weight);
    }

    public Integer getNode() {
        return node;
    }

    public Integer getWeight() {
        return weight;
    }

}

class IO {

    private static BufferedReader reader;
    private static BufferedWriter writer;

    public static void init() {
        reader = new BufferedReader(new InputStreamReader(System.in));
        writer = new BufferedWriter(new OutputStreamWriter(System.out));
    }

    public static void close() {
        try {
            writer.flush();
            writer.close();
            reader.close();
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }
    }

    public static String read() {
        String input = null;
        try {
            input = reader.readLine();
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }

        if (input == null) {
            throw new IllegalArgumentException();
        }

        return input;
    }

    public static void write(Object message) {
        try {
            writer.write(String.valueOf(message));
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }
    }
}

```

# 결론
다익스트라 알고리즘을 사용해 큰 어려움 없이 풀었으나, 메소드를 역할에 따라 나누고, 코드를 깨끗하게 작성하려고 하다 보니 시간이 걸렸다. 이번 문제에서는 n x n 배열과 Map, List를 각각 사용해보면서 List가 더 빠른 것을 체감할 수 있었다.
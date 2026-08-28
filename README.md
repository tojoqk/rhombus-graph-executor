rhombus-graph-executor
======================

Rhombus bindings for [graph-executor](https://github.com/tojoqk/graph-executor).

## Quick Start

![Water Jug Graph](examples/simple-water-jug.svg)

```rhombus
#lang rhombus/static

import lib("rhombus-graph-executor") open

class JugState(left :: Nat, right :: Nat)

fun jug_graph(g, left_cap, right_cap, target):
  when left_cap == right_cap
  | error(@str{jug_graph: must not be same caps (@left_cap, @right_cap)})

  fun pour_left_to_right_message(JugState(left, right)):
    def amount = math.min(left, right_cap - right)
    message(@str{Poured @amount gallons from @(left_cap)G to @(right_cap)G})

  fun pour_right_to_left_message(JugState(left, right)):
    def amount = math.min(right, left_cap - left)
    message(@str{Poured @amount gallons from @(right_cap)G to @(left_cap)G})

  fun prompt_playing(JugState(left, right)):
    @str{Goal: Make exactly @(target)G
           Current Status:
             [ @(left_cap)G Jug: @(left)/@(left_cap) | @(right_cap)G Jug: @(right)/@(right_cap) ]
           What will you do?}
  def make_node = node_maker(g)
  def playing = make_node("Playing", ~type: #'puzzle, ~prompt: code(prompt_playing))
  def check_clear = make_node("Check Clear", ~type: #'check_clear)
  def cleared = make_node(
    "Cleared!", ~type: #'terminal,
    ~before: code(fun (_): message(@str{Congratulations! You made exactly @target gallons!})),
  )

  def edges = [
    make_edge(
      @str{Fill @(left_cap)G}, ~from: playing, ~to: check_clear,
      ~when: code(fun (JugState(left, _)): left < left_cap),
      ~trans: code(fun (JugState(_, right)): JugState(left_cap, right)),
      ~before: code(fun (_): message(@str{Filled the @(left_cap)-gallon jug.})),
    ),
    make_edge(
      @str{Fill @(right_cap)G}, ~from: playing, ~to: check_clear,
      ~when: code(fun (JugState(_, right)): right < right_cap),
      ~trans: code(fun (JugState(left, _)): JugState(left, right_cap)),
      ~before: code(fun (_): message(@str{Filled the @(right_cap)-gallon jug.})),
    ),
    make_edge(
      @str{Empty @(left_cap)G}, ~from: playing, ~to: check_clear,
      ~when: code(fun (JugState(left, _)): 0 < left),
      ~trans: code(fun (JugState(_, right)): JugState(0, right)),
      ~before: code(fun (_): message(@str{Emptied the @(left_cap)-gallon jug.})),
    ),
    make_edge(
      @str{Empty @(right_cap)G}, ~from: playing, ~to: check_clear,
      ~when: code(fun (JugState(_, right)): right > 0),
      ~trans: code(fun (JugState(left, _)): JugState(left, 0)),
      ~before: code(fun (_): message(@str{Emptied the @(right_cap)-gallon jug.})),
    ),
    make_edge(
      @str{Pour @(left_cap)G -> @(right_cap)G}, ~from: playing, ~to: check_clear,
      ~when: code(fun (JugState(left, right)): left > 0 && right < right_cap),
      ~trans: code(
                fun (JugState(left, right)):
                  def amount = math.min(left, right_cap - right)
                  JugState(left - amount, right + amount)
              ),
      ~before: code(pour_left_to_right_message),
    ),
    make_edge(
      @str{Pour @(right_cap)G -> @(left_cap)G}, ~from: playing, ~to: check_clear,
      ~when: code(fun (JugState(left, right)): right > 0 && left < left_cap),
      ~trans: code(
                fun (JugState(left, right)):
                  def amount = math.min(right, left_cap - left)
                  JugState(left + amount, right - amount)
              ),
      ~before: code(pour_right_to_left_message),
    ),
    make_edge(
      "Not yet", ~mode: #'auto, ~from: check_clear, ~to: playing,
      ~when: code(fun (JugState(left, right)): left != target && right != target),
    ),
    make_edge(
      "Clear!", ~mode: #'auto, ~from: check_clear, ~to: cleared,
      ~when: code(fun (JugState(left, right)): left == target || right == target),
    ),
  ]

  values(make_graph(g, ~edges), playing)

fun make_model(left_cap, right_cap, target):
  model(
    fun ():
      def values(graph, node_init) = jug_graph("Water Jug Puzzle", left_cap, right_cap, target)
      values([graph], node_init, JugState(0, 0))
  )

module main:
  import rhombus/cmdline open

  def options:
    parse:
      ~init: { #'mode: #'dot }
      once_any:
        flag "--dot":
          ~help: "Generate dot"
          state[#'mode] := #'dot
        flag "--console":
          ~help: "Run console"
          state[#'mode] := #'console
        flag "--random":
          ~help: "Run random"
          state[#'mode] := #'random

  def m = make_model(3, 5, 4)
  def trace_display = #'hide
  match options[#'mode]
  | #'dot:
      def global = dot_global_config(~rankdir: #'LR)
      def config = dot_config(~global)
      render_dot(m, ~config)
  | #'console:
      def commands = [[#'quit, #'q, "Quit"]]
      def config = console_config(~commands, ~trace_display)
      console_run(m, ~config)
      #void
  | #'random:
      def chooser = fun (_): #'random
      def config = console_config(~chooser, ~trace_display)
      def j :: PairList.of(PairList) = console_run(m, ~config)
      def steps = for values(cnt = 0) (e in j):
        if e[0] == #'choose
        | cnt + 1
        | cnt
      println(@str{Solved in @steps steps!})

module test:
  def m = make_model(3, 5, 4)
  fun is_terminal_node(x :: NodeInfo): x.type == #'terminal
  def invariant = fun (_, JugState(l, r)): 0 <= l && l <= 3 && 0 <= r && r <= 5
  def is_goal = fun (n, _): is_terminal_node(n)
  check:
    find_livelock(m) ~is #false
    find_deadlock(m, is_terminal_node) ~is #false
    find_false_terminal(m, is_terminal_node) ~is #false
    find_auto_conflict(m) ~is #false
    find_counterexample(m, invariant) ~is #false
    find_witness(m, is_goal) ~is PairList[
      PairList[#'auto, PairList["Clear!"]],
      PairList[#'choose, PairList["Pour 5G -> 3G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Fill 5G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Pour 5G -> 3G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Empty 3G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Pour 5G -> 3G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Empty 3G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Fill 5G"]],
      PairList[#'auto, PairList["Not yet"]],
      PairList[#'choose, PairList["Fill 3G"]],
    ]
```

## License

Apache License 2.0. See [LICENSE](LICENSE) for details.

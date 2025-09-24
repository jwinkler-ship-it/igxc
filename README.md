import React, { useState } from "react";
import { DndProvider, useDrag, useDrop } from "react-dnd";
import { HTML5Backend } from "react-dnd-html5-backend";

const ItemTypes = { RUNNER: "runner" };

const teamsInfo = {
  Boscobel: { orgID: 46, hex: "#FF0000", text: "#FFFFFF" },
  Fennimore: { orgID: 126, hex: "#FFD700", text: "#000000" },
  "Iowa-Grant": { orgID: 173, hex: "#8B0000", text: "#FFFFFF" },
  Brookwood: { orgID: 54, hex: "#000000", text: "#FF0000" },
};

function getLogo(orgID) {
  return `https://schools.wiaawi.org/Directory/School/GetSchoolProfilePic?orgID=${orgID}`;
}

function Runner({ runner, index, moveRunner, teamPositions }) {
  const [, dragRef] = useDrag({ type: ItemTypes.RUNNER, item: { index } });
  const [, dropRef] = useDrop({
    accept: ItemTypes.RUNNER,
    hover: (item) => {
      if (item.index !== index) {
        moveRunner(item.index, index);
        item.index = index;
      }
    },
  });

  const teamNumber = teamPositions[runner.team][index];
  const teamInfo = teamsInfo[runner.team] || {};
  const logo = teamInfo.orgID ? getLogo(teamInfo.orgID) : null;
  const style = teamInfo.hex ? { backgroundColor: teamInfo.hex, color: teamInfo.text } : {};

  return (
    <div
      ref={(node) => dragRef(dropRef(node))}
      className="flex items-center justify-between p-2 mb-1 rounded shadow cursor-move"
      style={style}
    >
      <div className="flex items-center gap-2">
        {logo && <img src={logo} alt={`${runner.team} logo`} className="w-6 h-6" />}
        <span>
          {index + 1}. {runner.name} ({runner.team} #{teamNumber}) - {runner.time}
        </span>
      </div>
    </div>
  );
}

function calculateScores(runners) {
  const scores = {};
  runners.forEach((r, i) => {
    if (!scores[r.team]) scores[r.team] = [];
    scores[r.team].push(i + 1);
  });

  const results = {};
  for (const team in scores) {
    const sorted = scores[team].sort((a, b) => a - b);
    const scoring = sorted.slice(0, 5);
    const total = scoring.reduce((a, b) => a + b, 0);
    const displacers = sorted.slice(5, 7);
    results[team] = { total, scoring, displacers };
  }
  return results;
}

function calculateTeamPositions(runners) {
  const teamPositions = {};
  const counters = {};
  runners.forEach((r, idx) => {
    if (!counters[r.team]) counters[r.team] = 0;
    counters[r.team] += 1;
    if (!teamPositions[r.team]) teamPositions[r.team] = {};
    teamPositions[r.team][idx] = counters[r.team];
  });
  return teamPositions;
}

function calculateIndividualQualifiers(runners, sortedScores) {
  const topTeams = sortedScores.slice(0, 2).map(([team]) => team);
  const individualRunners = runners
    .map((r, i) => ({ ...r, place: i + 1 }))
    .filter((r) => !topTeams.includes(r.team));
  return individualRunners.slice(0, 5);
}

export default function App() {
  const [inputText, setInputText] = useState("");
  const [runners, setRunners] = useState([]);
  const [originalRunners, setOriginalRunners] = useState([]);
  const [inputCollapsed, setInputCollapsed] = useState(false);

  const parseInput = () => {
    const lines = inputText.split("\n");
    const parsed = lines
      .map((line) => {
        const parts = line.split("\t");
        if (parts.length >= 5) {
          return { name: parts[1].trim(), team: parts[3].trim(), time: parts[4].trim() };
        }
        return null;
      })
      .filter((r) => r !== null);
    setRunners(parsed);
    setOriginalRunners(parsed);
    setInputCollapsed(true); // collapse the input box after loading
  };

  const moveRunner = (fromIndex, toIndex) => {
    const updated = [...runners];
    const [moved] = updated.splice(fromIndex, 1);
    updated.splice(toIndex, 0, moved);
    setRunners(updated);
  };

  const resetRunners = () => {
    setRunners([...originalRunners]);
    setInputCollapsed(false); // expand input box on reset
  };

  const scores = calculateScores(runners);
  const sortedScores = Object.entries(scores).sort((a, b) => a[1].total - b[1].total);
  const teamPositions = calculateTeamPositions(runners);
  const individualQualifiers = calculateIndividualQualifiers(runners, sortedScores);

  return (
    <DndProvider backend={HTML5Backend}>
      <div className="p-6 bg-gray-100 min-h-screen">
        <h1 className="text-2xl font-bold mb-4">Cross Country Drag & Drop</h1>

        <textarea
          rows={inputCollapsed ? 2 : 10}
          className="w-full p-2 mb-4 border rounded transition-all duration-300"
          placeholder="Paste results from Milesplit here"
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
        />

        <div className="mb-4 flex gap-2">
          <button className="p-2 bg-blue-500 text-white rounded" onClick={parseInput}>
            Load Runners
          </button>
          <button className="p-2 bg-gray-500 text-white rounded" onClick={resetRunners}>
            Reset
          </button>
        </div>

        <div className="grid grid-cols-2 gap-6">
          <div className="overflow-y-auto max-h-[70vh]">
            <h2 className="font-semibold mb-2">Finish Order</h2>
            {runners.map((runner, i) => (
              <Runner key={i} runner={runner} index={i} moveRunner={moveRunner} teamPositions={teamPositions} />
            ))}
          </div>

          <div className="sticky top-0 h-[70vh] overflow-y-auto p-2 bg-gray-50 rounded shadow">
            <h2 className="font-semibold mb-2">Team Scores</h2>
            {sortedScores.map(([team, { total, scoring, displacers }]) => {
              const teamInfo = teamsInfo[team] || {};
              const logo = teamInfo.orgID ? getLogo(teamInfo.orgID) : null;
              const style = teamInfo.hex ? { backgroundColor: teamInfo.hex, color: teamInfo.text } : {};
              return (
                <div key={team} className="p-3 mb-2 rounded shadow flex items-center gap-2" style={style}>
                  {logo && <img src={logo} alt={`${team} logo`} className="w-6 h-6" />}
                  <strong>{team}</strong>: {total} (places {scoring.join(", ")}); Displacers: {displacers.join(", ")}
                </div>
              );
            })}

            <h2 className="font-semibold mt-4 mb-2">Individual Qualifiers</h2>
            <ol>
              {individualQualifiers.map((r, i) => (
                <li key={i}>
                  {r.name} ({r.team}) - Place {r.place}
                </li>
              ))}
            </ol>
          </div>
        </div>
      </div>
    </DndProvider>
  );
}

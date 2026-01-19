import { useState } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Plus, Trash2, Download } from "lucide-react";

// === Bibliothèque BET NF C 15-100 ===
const COMPONENTS = [
  { type: "prise", label: "Prise 16A", power: 3680, max: 8, section: "2.5 mm²", dj: "20A" },
  { type: "lampe", label: "Point lumineux", power: 100, max: 8, section: "1.5 mm²", dj: "16A" },
  { type: "tableau", label: "Tableau électrique" },
];

export default function App() {
  const [elements, setElements] = useState([]);
  const [connections, setConnections] = useState([]);
  const [selected, setSelected] = useState(null);

  // === Ajout composant ===
  const addElement = (type) => {
    setElements([...elements, { id: Date.now(), type, x: 100, y: 100 }]);
  };

  // === Drag & Drop ===
  const move = (id, dx, dy) => {
    setElements((els) => els.map((e) => e.id === id ? { ...e, x: e.x + dx, y: e.y + dy } : e));
  };

  const removeElement = (id) => {
    setElements(elements.filter((e) => e.id !== id));
    setConnections(connections.filter((c) => c.from !== id && c.to !== id));
  };

  // === Connexions ===
  const connect = (from, to) => {
    if (from && to && from !== to) {
      setConnections([...connections, { from, to }]);
    }
  };

  // === Circuits & calculs NF C 15-100 ===
  const circuits = {};
  connections.forEach((c) => {
    const el = elements.find((e) => e.id === c.to);
    const src = elements.find((e) => e.id === c.from);
    if (!el || !src || src.type !== "tableau") return;
    if (!circuits[c.from]) circuits[c.from] = [];
    circuits[c.from].push(el);
  });

  // === Export PDF simplifié ===
  const exportPDF = () => {
    const content = JSON.stringify(circuits, null, 2);
    const blob = new Blob([content], { type: "application/pdf" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "schema-electrique.pdf";
    a.click();
  };

  return (
    <div className="p-6 grid grid-cols-4 gap-4">
      {/* Palette */}
      <Card className="col-span-1">
        <CardContent className="p-4 space-y-2">
          <h2 className="text-lg font-semibold">Composants NF C 15-100</h2>
          {COMPONENTS.map((c) => (
            <Button key={c.type} className="w-full" onClick={() => addElement(c.type)}>
              <Plus size={16} /> {c.label}
            </Button>
          ))}
          <Button className="w-full mt-4" onClick={exportPDF}>
            <Download size={16} /> Export PDF
          </Button>
        </CardContent>
      </Card>

      {/* Zone de schéma */}
      <Card className="col-span-3">
        <CardContent className="p-4">
          <h2 className="text-lg font-semibold mb-2">Schéma électrique BET</h2>
          <svg width="100%" height="520" className="border rounded-xl">
            {/* Fils normalisés */}
            {connections.map((c, i) => {
              const from = elements.find((e) => e.id === c.from);
              const to = elements.find((e) => e.id === c.to);
              if (!from || !to) return null;
              return (
                <line
                  key={i}
                  x1={from.x + 80}
                  y1={from.y + 20}
                  x2={to.x}
                  y2={to.y + 20}
                  stroke="#000"
                  strokeWidth="2"
                />
              );
            })}

            {/* Symboles */}
            {elements.map((el) => (
              <g
                key={el.id}
                transform={`translate(${el.x}, ${el.y})`}
                onMouseDown={(e) => {
                  const startX = e.clientX;
                  const startY = e.clientY;
                  const moveHandler = (ev) => move(el.id, ev.clientX - startX, ev.clientY - startY);
                  document.addEventListener("mousemove", moveHandler);
                  document.addEventListener("mouseup", () => document.removeEventListener("mousemove", moveHandler), { once: true });
                }}
                onClick={() => selected ? (connect(selected, el.id), setSelected(null)) : setSelected(el.id)}
              >
                <rect width="80" height="40" rx="8" fill="#f4f4f5" stroke="#18181b" />
                <text x="40" y="25" textAnchor="middle" fontSize="11">{el.type}</text>
                <foreignObject x="60" y="-10" width="30" height="30">
                  <button onClick={() => removeElement(el.id)} className="text-red-500"><Trash2 size={14} /></button>
                </foreignObject>
              </g>
            ))}
          </svg>

          {/* Tableau de contrôle */}
          <div className="mt-4">
            <h3 className="font-semibold">Contrôle NF C 15-100</h3>
            {Object.values(circuits).map((els, i) => {
              const power = els.reduce((s, e) => s + (COMPONENTS.find(c => c.type === e.type)?.power || 0), 0);
              const rule = COMPONENTS.find(c => c.type === els[0]?.type);
              const ok = els.length <= (rule?.max || 999);
              return (
                <div key={i} className={`text-sm ${ok ? "text-green-600" : "text-red-600"}`}>
                  Circuit {i + 1} : {els.length} points | {power} W | {rule?.section} | DJ {rule?.dj}
                </div>
              );
            })}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
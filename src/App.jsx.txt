

import React from 'react';

export default function DashboardSLA() {
  // estados primeiro para evitar erro de execução
  const [motoristas, setMotoristas] = React.useState([]);
  const [entregasSet, setEntregasSet] = React.useState(new Set());
  const [recebimentoSet, setRecebimentoSet] = React.useState(new Set());
  const [pedidosPorBase, setPedidosPorBase] = React.useState({});
  const [saidaRows, setSaidaRows] = React.useState([]);
  const [modoTela, setModoTela] = React.useState('consulta');
  const [baseSelecionada, setBaseSelecionada] = React.useState('Todas as bases');
  const [sheetRecebimentoUrl, setSheetRecebimentoUrl] = React.useState('');
  const [sheetSaidaUrl, setSheetSaidaUrl] = React.useState('');
  const [sheetEntregasUrl, setSheetEntregasUrl] = React.useState('');
  const [sheetStatus, setSheetStatus] = React.useState('');
  const [historicoSla, setHistoricoSla] = React.useState([]);
  const API_APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbwoOvZ8g3epZQlR5y8sC5fIgwSWSCMkvaE6oby2_ndLciUGDf_SPhBnrezOMvsMvvcqtg/exec';

  // persistência local para manter os dados após recarregar a página no mesmo dispositivo
  React.useEffect(() => {
    const savedUrl = window.localStorage.getItem('dashboard_sheet_url');
    // URLs separadas do Google Sheets são preenchidas manualmente na aba de atualização
    const saved = window.localStorage.getItem('dashboard_sla_jt_data');
    if (!saved) return;
    try {
      const data = JSON.parse(saved);
      setMotoristas(data.motoristas || []);
      setEntregasSet(new Set(data.entregas || []));
      setRecebimentoSet(new Set(data.recebimento || []));
      setPedidosPorBase(data.pedidosPorBase || {});
      setSaidaRows(data.saidaRows || []);
    } catch (e) {
      console.error('Erro ao carregar dados salvos', e);
    }
  }, []);

  React.useEffect(() => {
    window.localStorage.setItem(
      'dashboard_sla_jt_data',
      JSON.stringify({
        motoristas,
        entregas: Array.from(entregasSet),
        recebimento: Array.from(recebimentoSet),
        pedidosPorBase,
        saidaRows,
      })
    );
  }, [motoristas, entregasSet, recebimentoSet, pedidosPorBase, saidaRows]);

  const carregarViaAPI = async () => {
    try {
      setSheetStatus('Carregando dados da API central...');
      const response = await fetch(API_APPS_SCRIPT_URL, {
        method: 'GET',
        redirect: 'follow',
        headers: { 'Accept': 'application/json,text/plain,*/*' }
      });

      const rawText = await response.text();
      let data;
      try {
        data = JSON.parse(rawText);
      } catch (parseError) {
        setSheetStatus(`API respondeu formato inválido: ${rawText.slice(0, 120)}...`);
        console.error('Resposta inválida da API', rawText);
        return;
      }

      const rowsRec = data.recebimento || [];
      const rowsSai = data.saida || [];
      const rowsEnt = data.entregas || [];

      const recebidosSet = new Set();
      const baseMap = {};
      rowsRec.forEach((row) => {
        const pedido = row['Número de pedido JMS'] || row['Remessa'] || row['A'];
        if (!pedido) return;
        recebidosSet.add(String(pedido));
        baseMap[String(pedido)] = normalizarBase(row['目的网点网点'] || row['Base']);
      });

      const entreguesTemp = new Set(
        rowsEnt.map((r) => String(r['Número de pedido JMS'] || r['A'] || '')).filter(Boolean)
      );

      const saidaNormalizada = rowsSai.map((row) => ({
        'Número de pedido JMS': String(row['Número de pedido JMS'] || row['A'] || ''),
        'Correio de coleta ou entrega': row['Correio de coleta ou entrega'] || row['motorista'] || '',
      })).filter((r) => r['Número de pedido JMS']);

      setRecebimentoSet(recebidosSet);
      setPedidosPorBase(baseMap);
      setEntregasSet(entreguesTemp);
      setSaidaRows(saidaNormalizada);
      setSheetStatus(`API carregada com sucesso • Rec: ${recebidosSet.size} | Sai: ${saidaNormalizada.length} | Ent: ${entreguesTemp.size}`);
    } catch (error) {
      setSheetStatus(`Erro ao carregar API Apps Script: ${error?.message || 'falha desconhecida'}`);
      console.error(error);
    }
  };

  const carregarGoogleSheets = async () => {
    if (!sheetRecebimentoUrl || !sheetSaidaUrl || !sheetEntregasUrl) {
      setSheetStatus('Informe as 3 URLs CSV (Recebimento, Saída e Entregas)');
      return;
    }

    try {
      setSheetStatus('Carregando dados do Google Sheets...');
      const [respRec, respSai, respEnt] = await Promise.all([
        fetch(sheetRecebimentoUrl),
        fetch(sheetSaidaUrl),
        fetch(sheetEntregasUrl),
      ]);

      const [csvRec, csvSai, csvEnt] = await Promise.all([
        respRec.text(),
        respSai.text(),
        respEnt.text(),
      ]);

      const parseCsv = (csv) => {
        const linhas = String(csv).split(/\r?\n/).filter((l) => String(l).trim());
        if (!linhas.length) return [];

        const limpar = (valor) =>
          String(valor || '')
            .replace(/^\uFEFF/, '')
            .replace(/^\"|\"$/g, '')
            .trim();

        const delimiter = linhas[0].includes(';') ? ';' : ',';
        const headers = linhas[0].split(delimiter).map((h) => limpar(h));

        return linhas.slice(1).map((linha) => {
          const cols = linha.split(delimiter).map((c) => limpar(c));
          return Object.fromEntries(headers.map((h, i) => [h, cols[i] || '']));
        });
      };

      const rowsRec = parseCsv(csvRec);
      const rowsSai = parseCsv(csvSai);
      const rowsEnt = parseCsv(csvEnt);

      const getCampo = (row, candidatos) => {
        const keys = Object.keys(row || {});
        const found = keys.find((k) =>
          candidatos.some((c) => String(k).trim().toLowerCase() === String(c).trim().toLowerCase())
        );
        return found ? row[found] : undefined;
      };

      const recebidosSet = new Set();
      const baseMap = {};
      rowsRec.forEach((row) => {
        const pedido = getCampo(row, ['Número de pedido JMS', 'A', 'Remessa', 'numero de pedido jms']);
        if (!pedido) return;
        recebidosSet.add(pedido);
        baseMap[pedido] = normalizarBase(getCampo(row, ['目的网点网点', 'base', 'Base']));
      });

      const entreguesTemp = new Set(
        rowsEnt.map((r) => getCampo(r, ['Número de pedido JMS', 'A', 'numero de pedido jms'])).filter(Boolean)
      );

      const saidaNormalizada = rowsSai.map((row) => ({
        'Número de pedido JMS': getCampo(row, ['Número de pedido JMS', 'A', 'numero de pedido jms', 'pedido']),
        'Correio de coleta ou entrega': getCampo(row, ['Correio de coleta ou entrega', 'motorista', 'R', 'correio de coleta ou entrega']),
      })).filter((r) => r['Número de pedido JMS']);

      setRecebimentoSet(recebidosSet);
      setPedidosPorBase(baseMap);
      setEntregasSet(entreguesTemp);
      setSaidaRows(saidaNormalizada);
      if (!recebidosSet.size && !saidaNormalizada.length && !entreguesTemp.size) {
        setSheetStatus(`Nenhum dado encontrado • Verifique os cabeçalhos e se as abas estão publicadas em CSV. Headers Rec: ${Object.keys(rowsRec[0] || {}).join(' | ')}`);
      } else {
        setSheetStatus(`Dados carregados com sucesso • Rec: ${recebidosSet.size} | Sai: ${saidaNormalizada.length} | Ent: ${entreguesTemp.size}`);
      }
    } catch (error) {
      setSheetStatus('Erro ao carregar Google Sheets');
      console.error(error);
    }
  };

  const limparDados = () => {
    setMotoristas([]);
    setEntregasSet(new Set());
    setRecebimentoSet(new Set());
    setPedidosPorBase({});
    setBaseSelecionada('Todas as bases');
    setSaidaRows([]);
  };

  const normalizarBase = (valor) => String(valor || '').trim().toUpperCase().replace(/\s+/g, ' ');

  const bases = React.useMemo(() => {
    const basesArquivo = Array.from(
      new Set(Object.values(pedidosPorBase).map((b) => normalizarBase(b)).filter(Boolean))
    ).sort();
    return ['Todas as bases', ...basesArquivo];
  }, [pedidosPorBase]);

  // indicadores derivados
  const recebidos = React.useMemo(() => {
    if (baseSelecionada === 'Todas as bases') return recebimentoSet.size;
    return Object.values(pedidosPorBase).filter(
      (base) => String(base).trim() === String(baseSelecionada).trim()
    ).length;
  }, [recebimentoSet, pedidosPorBase, baseSelecionada]);

  const expedidosCruzados = motoristas.reduce((acc, m) => acc + m.expedidos, 0);
  const entregues = motoristas.reduce((acc, m) => acc + m.entregues, 0);
  const pendentes = recebidos - entregues;
  const sla = recebidos ? ((entregues / recebidos) * 100).toFixed(1) : '0.0';

  React.useEffect(() => {
    const hora = new Date().toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' });
    setHistoricoSla((prev) => {
      const novo = [...prev, { hora, valor: Number(sla) }].slice(-12);
      return novo;
    });
  }, [sla]);

  React.useEffect(() => {
    // atualização automática pela API central a cada 15 minutos
    carregarViaAPI();
    const intervalo = setInterval(() => {
      carregarViaAPI();
    }, 15 * 60 * 1000);

    return () => clearInterval(intervalo);
  }, []);

  const comparativo = [
    { indicador: 'Recebidos', quantidade: recebidos },
    { indicador: 'Expedidos', quantidade: expedidosCruzados },
    { indicador: 'Entregues', quantidade: entregues },
    { indicador: 'Pendentes', quantidade: pendentes },
  ];

  // Dados validados a partir da planilha de saída
  // Coluna R = "Correio de coleta ou entrega"

  const processarEntregas = async (event) => {
    const file = event.target.files?.[0];
    if (!file) return;

    const XLSX = await import('xlsx');
    const buffer = await file.arrayBuffer();
    const workbook = XLSX.read(buffer, { type: 'array' });
    const sheet = workbook.Sheets[workbook.SheetNames[0]];
    const rows = XLSX.utils.sheet_to_json(sheet);

    const entregues = new Set(
      rows.map((row) => row['Número de pedido JMS'] || row['A']).filter(Boolean)
    );

    setEntregasSet(entregues);
  };

  React.useEffect(() => {
    if (!saidaRows.length || !recebimentoSet.size) {
      setMotoristas([]);
      return;
    }

    const agrupado = {};
    const pedidosProcessados = new Set();

    saidaRows.forEach((row) => {
      const nome = row['Correio de coleta ou entrega'];
      const pedido = row['Número de pedido JMS'] || row['A'];
      if (!nome || !pedido || !recebimentoSet.has(pedido)) return;
      const basePedido = String(pedidosPorBase[pedido] || '').trim();
      const baseFiltro = String(baseSelecionada).trim();
      if (baseFiltro !== 'Todas as bases' && basePedido !== baseFiltro) return;
      if (pedidosProcessados.has(pedido)) return;
      pedidosProcessados.add(pedido);

      if (!agrupado[nome]) {
        agrupado[nome] = {
          nome,
          expedidos: 0,
          entregues: 0,
          pendentes: 0,
          progresso: 0,
        };
      }

      agrupado[nome].expedidos += 1;
      if (entregasSet.has(pedido)) {
        agrupado[nome].entregues += 1;
      }
    });

    const lista = Object.values(agrupado).map((m) => ({
      ...m,
      pendentes: m.expedidos - m.entregues,
      progresso: m.expedidos ? ((m.entregues / m.expedidos) * 100).toFixed(1) : '0.0',
    }));

    setMotoristas(lista);
  }, [saidaRows, entregasSet, recebimentoSet, pedidosPorBase, baseSelecionada]);

  const processarArquivo = async (event) => {
    const file = event.target.files?.[0];
    if (!file) return;

    const XLSX = await import('xlsx');
    const buffer = await file.arrayBuffer();
    const workbook = XLSX.read(buffer, { type: 'array' });
    const sheet = workbook.Sheets[workbook.SheetNames[0]];
    const rows = XLSX.utils.sheet_to_json(sheet);

    setSaidaRows(rows);
  };

  const totalExpedidosMotoristas = motoristas.reduce((acc, m) => acc + m.expedidos, 0);
  const totalEntreguesMotoristas = motoristas.reduce((acc, m) => acc + m.entregues, 0);
  const totalProgresso = totalExpedidosMotoristas ? ((totalEntreguesMotoristas / totalExpedidosMotoristas) * 100).toFixed(1) : '0.0';

  return (
    <div className="min-h-screen bg-gradient-to-br from-red-800 via-red-600 to-red-400 p-6">
      <div className="max-w-7xl mx-auto space-y-6">
        <div className="flex flex-col md:flex-row md:items-center md:justify-between gap-4">
          <div className="bg-white/95 backdrop-blur rounded-3xl shadow-2xl px-6 py-5 border border-white/30">
            <div className="flex items-center gap-4">
              <div className="bg-white rounded-2xl px-3 py-2 shadow-lg border border-red-100 flex items-center">
                <div className="flex items-center gap-2 px-2">
                  <div className="text-red-600 font-black text-2xl tracking-tight">J&amp;T</div>
                  <div className="text-gray-700 font-semibold text-sm leading-tight">
                    <div>EXPRESS</div>
                    <div className="text-xs text-gray-500">Brasil</div>
                  </div>
                </div>
              </div>
              <div>
                <h1 className="text-3xl font-bold text-gray-900">Dashboard SLA - Bases Logísticas</h1>
                <p className="text-gray-600">Painel executivo operacional • J&T Express Brasil</p>
              </div>
            </div>
          </div>
          <div className="flex gap-3 items-start">
            <div className="bg-white rounded-2xl shadow-sm p-1 flex gap-1">
              <button
                onClick={() => setModoTela('consulta')}
                className={`px-4 py-2 rounded-xl text-sm font-medium ${modoTela === 'consulta' ? 'bg-red-600 text-white' : 'text-gray-600'}`}
              >
                Visão Consulta
              </button>
              <button
                onClick={() => setModoTela('atualizacao')}
                className={`px-4 py-2 rounded-xl text-sm font-medium ${modoTela === 'atualizacao' ? 'bg-red-600 text-white' : 'text-gray-600'}`}
              >
                Atualização de Dados
              </button>
            </div>
            <select value={baseSelecionada} onChange={(e) => setBaseSelecionada(e.target.value)} className="border rounded-xl px-4 py-2 bg-white shadow-sm">
              {bases.map((base, i) => <option key={i}>{base}</option>)}
            </select>
          </div>
        </div>

        {modoTela === 'atualizacao' && (
          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-6">
            <h2 className="text-xl font-semibold mb-4">Central de Atualização de Dados</h2>
            <p className="text-sm text-gray-500 mb-4">Upload manual e atualização automática via Google Sheets.</p>

            <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
              <input value={sheetRecebimentoUrl} onChange={(e)=>setSheetRecebimentoUrl(e.target.value)} placeholder="URL CSV - Recebimento" className="border rounded-xl px-4 py-3" />
              <input value={sheetSaidaUrl} onChange={(e)=>setSheetSaidaUrl(e.target.value)} placeholder="URL CSV - Saída" className="border rounded-xl px-4 py-3" />
              <input value={sheetEntregasUrl} onChange={(e)=>setSheetEntregasUrl(e.target.value)} placeholder="URL CSV - Entregas" className="border rounded-xl px-4 py-3" />
            </div>
            <div className="flex gap-3 mb-6">
              <button onClick={carregarViaAPI} className="px-4 py-2 rounded-xl bg-red-700 text-white font-medium">Atualizar via API</button>
              <button onClick={carregarGoogleSheets} className="px-4 py-2 rounded-xl bg-red-600 text-white font-medium">Atualizar via Sheets</button>
              <button onClick={limparDados} className="px-4 py-2 rounded-xl border text-red-600 font-medium">Limpar Dados</button>
            </div>
            <div className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white mb-4">
              <p className="font-medium mb-2">Arquivo JSON da API (backup sem CORS)</p>
              <input type="file" accept=".json,.txt" onChange={async (event) => {
                const file = event.target.files?.[0];
                if (!file) return;
                try {
                  const text = await file.text();
                  const data = JSON.parse(text);
                  const rowsRec = data.recebimento || [];
                  const rowsSai = data.saida || [];
                  const rowsEnt = data.entregas || [];

                  const recebidosSet = new Set();
                  const baseMap = {};
                  rowsRec.forEach((row) => {
                    const pedido = String(row['Número de pedido JMS'] || row['Remessa'] || row['A'] || '');
                    if (!pedido) return;
                    recebidosSet.add(pedido);
                    baseMap[pedido] = normalizarBase(row['目的网点网点'] || row['Base']);
                  });

                  const entreguesTemp = new Set(
                    rowsEnt.map((r) => String(r['Número de pedido JMS'] || r['A'] || '')).filter(Boolean)
                  );

                  const saidaNormalizada = rowsSai.map((row) => ({
                    'Número de pedido JMS': String(row['Número de pedido JMS'] || row['A'] || ''),
                    'Correio de coleta ou entrega': row['Correio de coleta ou entrega'] || row['motorista'] || '',
                  })).filter((r) => r['Número de pedido JMS']);

                  setRecebimentoSet(recebidosSet);
                  setPedidosPorBase(baseMap);
                  setEntregasSet(entreguesTemp);
                  setSaidaRows(saidaNormalizada);
                  setSheetStatus(`JSON carregado com sucesso • Rec: ${recebidosSet.size} | Sai: ${saidaNormalizada.length} | Ent: ${entreguesTemp.size}`);
                } catch (e) {
                  setSheetStatus('Erro ao carregar arquivo JSON');
                }
              }} className="w-full border rounded-xl px-4 py-2 bg-white shadow-sm" title="Upload JSON" />
            </div>

            <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
              <div className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white">
                <p className="font-medium mb-2">Arquivo de Saída</p>
                <input type="file" accept=".xlsx,.xls,.csv" onChange={processarArquivo} className="w-full border rounded-xl px-4 py-2 bg-white shadow-sm" title="Upload saída" />
              </div>
              <div className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white">
                <p className="font-medium mb-2">Arquivo de Entregas</p>
                <input type="file" accept=".xlsx,.xls,.csv" onChange={processarEntregas} className="w-full border rounded-xl px-4 py-2 bg-white shadow-sm" title="Upload entregas" />
              </div>
              <div className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white">
                <p className="font-medium mb-2">Arquivo de Recebimento</p>
                <input type="file" accept=".xlsx,.xls,.csv" onChange={async (event) => { const file = event.target.files?.[0]; if (!file) return; const XLSX = await import('xlsx'); const buffer = await file.arrayBuffer(); const workbook = XLSX.read(buffer, { type: 'array' }); const sheet = workbook.Sheets[workbook.SheetNames[0]]; const rows = XLSX.utils.sheet_to_json(sheet); const recebidosSet = new Set(rows.map((row) => row['Número de pedido JMS'] || row['A'] || row['Remessa']).filter(Boolean)); const baseMap = {};
                rows.forEach((row) => {
                  const pedido = row['Número de pedido JMS'] || row['A'] || row['Remessa'];
                  const base = normalizarBase(row['目的网点网点']);
                  if (pedido) {
                    baseMap[pedido] = base;
                  }
                });
                setPedidosPorBase(baseMap);
                setRecebimentoSet(recebidosSet); }} className="w-full border rounded-xl px-4 py-2 bg-white shadow-sm" title="Upload recebimento" />
              </div>
            </div>
          </div>
        )}

        {sheetStatus && <div className="bg-white rounded-2xl px-4 py-3 shadow text-sm text-gray-600">{sheetStatus}</div>}

        <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-5"><p className="text-sm text-gray-500">Recebidos</p><p className="text-3xl font-bold">{recebidos}</p></div>
          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-5"><p className="text-sm text-gray-500">Expedidos</p><p className="text-3xl font-bold">{expedidosCruzados}</p></div>
          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-5"><p className="text-sm text-gray-500">Entregues</p><p className="text-3xl font-bold">{entregues}</p></div>
          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-5"><p className="text-sm text-gray-500">SLA</p><p className="text-3xl font-bold">{sla}%</p></div>
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-6">
            <h2 className="text-xl font-semibold mb-4">Comparativo Geral</h2>
            <div className="space-y-4">
              {comparativo.map((item, i) => {
                const percentual = recebidos ? ((item.quantidade / recebidos) * 100).toFixed(1) : '0.0';
                return (
                  <div key={i} className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white">
                    <div className="flex justify-between mb-2">
                      <span className="font-medium">{item.indicador}</span>
                      <span className="font-bold">{item.quantidade}</span>
                    </div>
                    <div className="w-full bg-gray-200 rounded-full h-3">
                      <div className="bg-gradient-to-r from-red-600 to-red-400 h-3 rounded-full" style={{ width: `${Math.min(Number(percentual), 100)}%` }} />
                    </div>
                    <p className="text-sm text-gray-500 mt-1">{percentual}% da base recebida</p>
                  </div>
                );
              })}
            </div>
          </div>

          <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-6">
            <h2 className="text-xl font-semibold mb-4">Distribuição</h2>
            <div className="space-y-4">
              {comparativo.slice(1).map((item, i) => {
                const percentual = recebidos ? ((item.quantidade / recebidos) * 100).toFixed(1) : '0.0';
                return (
                  <div key={i} className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white">
                    <div className="flex justify-between mb-2">
                      <span>{item.indicador}</span>
                      <span className="font-semibold">{percentual}%</span>
                    </div>
                    <div className="w-full bg-gray-200 rounded-full h-3">
                      <div className="bg-gradient-to-r from-red-500 to-red-300 h-3 rounded-full" style={{ width: `${Math.min(Number(percentual), 100)}%` }} />
                    </div>
                  </div>
                );
              })}
            </div>
          </div>
        </div>

        <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-6">
          <div className="flex items-center justify-between mb-4">
            <h2 className="text-xl font-semibold">Evolução SLA Hora a Hora</h2>
            <div className="text-sm text-gray-500">Meta: <span className="font-semibold text-green-600">95%</span></div>
          </div>
          {historicoSla.length === 0 ? (
            <p className="text-sm text-gray-500">Aguardando histórico...</p>
          ) : (
            <div className="space-y-3">
              <div className="relative h-64 border rounded-2xl p-4 bg-gray-50 overflow-hidden">
                <div className="absolute left-0 right-0 border-t-2 border-dashed border-green-500" style={{ top: '5%' }} />
                <div className="absolute right-2" style={{ top: '0%' }}>
                  <span className="text-xs font-semibold text-green-600">95%</span>
                </div>
                <svg viewBox="0 0 100 100" className="w-full h-full">
                  <polyline
                    fill="none"
                    stroke="#dc2626"
                    strokeWidth="2"
                    points={historicoSla.map((item, i) => `${(i / Math.max(historicoSla.length - 1, 1)) * 100},${100 - Math.min(item.valor, 100)}`).join(' ')}
                  />
                  {historicoSla.map((item, i) => (
                    <circle
                      key={i}
                      cx={(i / Math.max(historicoSla.length - 1, 1)) * 100}
                      cy={100 - Math.min(item.valor, 100)}
                      r="1.8"
                      fill="#dc2626"
                    />
                  ))}
                </svg>
              </div>
              <div className="grid grid-cols-6 gap-2 text-xs text-gray-500">
                {historicoSla.map((item, i) => (
                  <div key={i} className="text-center">{item.hora}</div>
                ))}
              </div>
            </div>
          )}
        </div>

        <div className="bg-white rounded-3xl shadow-2xl border border-white/40 p-6">
          <h2 className="text-xl font-semibold mb-4">Pacotes por Motorista (coluna R - Correio de coleta ou entrega)</h2>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div className="space-y-4">
              {motoristas.map((m, i) => (
                <div key={i} className="rounded-2xl border border-gray-100 shadow-sm p-4 bg-white">
                  <div className="flex justify-between mb-2">
                    <span className="font-semibold">{m.nome}</span>
                    <span className="text-sm">{m.entregues}/{m.expedidos}</span>
                  </div>
                  <div className="w-full bg-gray-200 rounded-full h-3">
                    <div className="bg-gradient-to-r from-red-600 to-red-400 h-3 rounded-full" style={{ width: `${m.progresso}%` }} />
                  </div>
                  <div className="flex justify-between text-sm text-gray-600 mt-2">
                    <span>Exp: {m.expedidos}</span>
                    <span>Ent: {m.entregues}</span>
                    <span>Pend: {m.pendentes}</span>
                    <span>{m.progresso}%</span>
                  </div>
                </div>
              ))}
            </div>
          </div>

          <div className="mt-6 rounded-2xl border border-gray-100 shadow-sm p-4 bg-white bg-gray-50">
            <div className="flex justify-between mb-2">
              <span className="font-semibold">Total Geral Motoristas</span>
              <span>{totalEntreguesMotoristas}/{totalExpedidosMotoristas}</span>
            </div>
            <div className="w-full bg-gray-200 rounded-full h-4">
              <div className="bg-gradient-to-r from-red-600 to-red-400 h-4 rounded-full" style={{ width: `${totalProgresso}%` }} />
            </div>
            <div className="flex justify-between text-sm text-gray-600 mt-2">
              <span>Expedidos: {totalExpedidosMotoristas}</span>
              <span>Entregues: {totalEntreguesMotoristas}</span>
              <span>{totalProgresso}%</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

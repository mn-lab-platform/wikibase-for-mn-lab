# MediaWiki:Common.js

Should be inserted into [MediaWiki:Common.js](https://thesaurus.mn.cenagis.edu.pl/wiki/MediaWiki:Common.js) on the wiki.
```javascript
const DICT_LABELS = {
    pl: 'Powrót do słownika: ',
    en: 'Back to dictionary: ',
    de: 'Zurück zum Wörterbuch: ',
    fr: 'Retour au dictionnaire: ',
    el: 'Επιστροφή στο λεξικό: '
};

function getPrefix() {
    var lang = (typeof mw !== 'undefined' && mw.config) ? mw.config.get('wgUserLanguage') : null;
    return DICT_LABELS[lang] || DICT_LABELS.en;
}

function renderBackButton() {
    var params = new URLSearchParams(window.location.search);
    var returnTo = params.get('returnTo');
    var returnLabel = params.get('returnLabel');
    if (!returnTo) return;
    if ($('#dict-back-btn').length > 0) return;

    var $btn = $('<a>')
        .attr('id', 'dict-back-btn')
        .attr('href', returnTo)
        .text(getPrefix() + returnLabel)
        .css({
            'display': 'inline-block',
            'margin': '10px 0',
            'padding': '6px 15px',
            'background': '#3b82f6',
            'color': '#ffffff',
            'border-radius': '4px',
            'text-decoration': 'none',
            'font-weight': '500'
        });

    $('#firstHeading').after($btn);
}

$(document).ready(function() {
    renderBackButton();

    var $app = $('#wiki-tree-app');
    if ($app.length === 0) return;

    var selectedQ = $app.attr('data-dict');
    if (!selectedQ) return;

    let BINDINGS = [];

    const MY_QUERY = 'PREFIX wd: <https://thesaurus.mn.cenagis.edu.pl/entity/> ' +
        'PREFIX wdt: <https://thesaurus.mn.cenagis.edu.pl/prop/direct/> ' +
        'PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#> ' +
        'PREFIX skos: <http://www.w3.org/2004/02/skos/core#> ' +
        'PREFIX schema: <http://schema.org/> ' +
        'SELECT DISTINCT ?root ?rootLabel ?parent ?child ?childLabel ' +
          '(GROUP_CONCAT(DISTINCT ?alt; separator="||") AS ?altLabels) ' +
          '(GROUP_CONCAT(DISTINCT ?desc; separator="||") AS ?descriptions) WHERE { ' +
          'BIND(wd:' + selectedQ + ' AS ?root) ' +
          '?child (wdt:P2|wdt:P20)+ ?root . ' +

          'OPTIONAL { ?child wdt:P2 ?p2 . FILTER(?p2 = ?root || EXISTS { ?p2 (wdt:P2|wdt:P20)+ ?root }) } ' +
          'OPTIONAL { ?child wdt:P20 ?p20 . FILTER(?p20 = ?root || EXISTS { ?p20 (wdt:P2|wdt:P20)+ ?root }) } ' +

          'BIND(COALESCE(?p2, ?p20) AS ?parent) ' +
          'FILTER(BOUND(?parent)) ' +

          'OPTIONAL { ?root rdfs:label ?rootL . FILTER(lang(?rootL) = "en") } ' +
          'OPTIONAL { ?child rdfs:label ?childL . FILTER(lang(?childL) = "en") } ' +
          'OPTIONAL { ?child skos:altLabel ?alt . FILTER(lang(?alt) = "en") } ' +
          'OPTIONAL { ?child schema:description ?desc . FILTER(lang(?desc) = "en") } ' +
          'BIND(COALESCE(?rootL, STR(?root)) AS ?rootLabel) ' +
          'BIND(COALESCE(?childL, STR(?child)) AS ?childLabel) ' +
        '} ' +
        'GROUP BY ?root ?rootLabel ?parent ?child ?childLabel ' +
        'ORDER BY ?parent ?childLabel';

    function initDictionary(sparqlQuery) {
        const sparqlEndpoint = '/sparql?query=' + encodeURIComponent(sparqlQuery) + '&format=json';

        fetch(sparqlEndpoint)
            .then(function(res) { 
                if (!res.ok) throw new Error("HTTP error! Status: " + res.status); 
                return res.json(); 
            })
            .then(function(data) {
                if (data && data.results && data.results.bindings && data.results.bindings.length > 0) {
                    BINDINGS = data.results.bindings;
                    buildTree(BINDINGS);
                    
                    var $btn = $('<button>')
                        .attr('id', 'download-btn')
                        .text('Download selected as Arches format')
                        .css({
                            'padding': '6px 15px',
                            'background': '#3b82f6',
                            'color': 'white',
                            'border': 'none',
                            'border-radius': '4px',
                            'cursor': 'pointer',
                            'font-weight': '500'
                        });
                    $('#download-btn-placeholder').html($btn);
                } else {
                    $('#tree-container').text("No data available for dictionary " + selectedQ + ".");
                }
            })
            .catch(function(err) {
                $('#tree-container').html('<span style="color:#ef4444; font-weight:500;">SPARQL Error: ' + err.message + '</span>');
            });
    }

    function escapeXml(str) {
        return String(str)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&apos;');
    }

    function buildTree(bindings) {
        const container = document.getElementById('tree-container');
        if (!container) return;
        container.innerHTML = '';
        
        const getQ = function(uri) { return uri.split('/').pop(); };
        const rootUri = bindings[0].root.value;
        const rootLabel = (bindings[0].rootLabel && bindings[0].rootLabel.value) ? bindings[0].rootLabel.value : "Root Scheme";

        const nodes = {};
        bindings.forEach(function(b) {
            const cUri = b.child.value;
            const cLabel = (b.childLabel && b.childLabel.value) ? b.childLabel.value : cUri.split('/').pop();
            const pUri = b.parent.value;

            if (!nodes[cUri]) {
                nodes[cUri] = { uri: cUri, label: cLabel, parent: pUri, children: [] };
            }
        });

        const rootChildren = [];
        Object.values(nodes).forEach(function(node) {
            if (node.parent === rootUri) {
                rootChildren.push(node);
            } else if (nodes[node.parent]) {
                nodes[node.parent].children.push(node);
            }
        });

        const sortNodes = function(arr) {
            arr.sort(function(a, b) { return a.label.localeCompare(b.label); });
            arr.forEach(function(n) { if (n.children.length > 0) sortNodes(n.children); });
        };
        sortNodes(rootChildren);

        function renderNodes(nodeList) {
            const ul = document.createElement('ul');
            ul.style.listStyleType = 'none';
            ul.style.paddingLeft = '20px';
            
            nodeList.forEach(function(node) {
                const li = document.createElement('li');
                li.className = 'tree-node';
                const hasChildren = node.children.length > 0;
                const cleanLabel = node.label.split(' (said to be the same)')[0];

                li.innerHTML = '<div class="node-row" style="display: flex; align-items: center; gap: 5px; margin: 4px 0;">' +
                        (hasChildren ? '<span class="toggle-btn" style="cursor:pointer; user-select:none; width:15px; display:inline-block;">▼</span>' : '<span class="toggle-btn" style="visibility:hidden; width:15px; display:inline-block;">•</span>') +
                        '<input type="checkbox" class="node-cb" data-uri="' + node.uri + '" data-label="' + escapeXml(cleanLabel) + '">' +
                        '<span class="' + (hasChildren ? 'folder-click' : '') + '" style="' + (hasChildren ? 'cursor:pointer; font-weight:500;' : '') + '">' + escapeXml(node.label) + '</span>' +
                        '<small style="margin-left: 8px;"><a href="/wiki/Item:' + getQ(node.uri) + '?returnTo=' + encodeURIComponent(window.location.pathname + window.location.search) + '&amp;returnLabel=' + encodeURIComponent(rootLabel) + '" class="node-link" style="color: #2563eb;">[' + getQ(node.uri) + ']</a></small>' +
                    '</div>';

                if (hasChildren) {
                    li.appendChild(renderNodes(node.children));
                }
                ul.appendChild(li);
            });
            return ul;
        }

        const rootUl = document.createElement('ul');
        rootUl.style.listStyleType = 'none';
        rootUl.style.paddingLeft = '0px';
        rootUl.style.marginLeft = '0px';
        
        const rootLi = document.createElement('li');
        rootLi.className = 'tree-node root-node';
        rootLi.innerHTML = '<div class="node-row" style="display: flex; align-items: center; gap: 5px; margin-bottom: 8px;">' +
                '<span class="toggle-btn" style="cursor:pointer; user-select:none; width:15px; display:inline-block;">▼</span>' +
                '<input type="checkbox" class="lvl0">' +
                '<span class="folder-click"><strong>' + rootLabel + '</strong></span>' +
                '<small style="margin-left: 8px;"><a href="/wiki/Item:' + getQ(rootUri) + '?returnTo=' + encodeURIComponent(window.location.pathname + window.location.search) + '&amp;returnLabel=' + encodeURIComponent(rootLabel) + '" class="node-link" style="color: #2563eb;">[' + getQ(rootUri) + ']</a></small>' +
            '</div>';
        
        if (rootChildren.length > 0) {
            rootLi.appendChild(renderNodes(rootChildren));
        }
        rootUl.appendChild(rootLi);
        container.appendChild(rootUl);

        $(container).find('.toggle-btn, .folder-click').on('click', function() {
            var $node = $(this).closest('.tree-node');
            $node.find('> ul').toggle();
            var $btn = $node.find('> .node-row .toggle-btn');
            if ($btn.length && $btn.css('visibility') !== 'hidden') {
                $btn.text($node.find('> ul').is(':visible') ? '▼' : '►');
            }
        });

        $(container).find('input[type="checkbox"]').on('change', function() {
            var checked = $(this).prop('checked');
            var $li = $(this).closest('li');
            $li.find('input[type="checkbox"]').prop('checked', checked);
            if (checked) $li.find('> ul').show();
        });
    }

    function collectAncestors(uri, bindings, checkedSet) {
        bindings.forEach(function(b) {
            if (b.child.value === uri) {
                const parentUri = b.parent.value;
                if (parentUri !== b.root.value && !checkedSet.has(parentUri)) {
                    checkedSet.add(parentUri);
                    collectAncestors(parentUri, bindings, checkedSet);
                }
            }
        });
    }

    $(document).off('click', '#download-btn').on('click', '#download-btn', function() {
        const checkedUris = new Set();
        $('input[type="checkbox"]:checked').each(function() {
            const uri = $(this).attr('data-uri');
            if (uri) checkedUris.add(uri);
        });

        if (checkedUris.size === 0) return alert('Select elements to export.');

        const allExportUris = new Set(checkedUris);
        allExportUris.forEach(function(uri) { collectAncestors(uri, BINDINGS, allExportUris); });

        const schemeUri = BINDINGS[0].root.value;
        const schemeLabel = (BINDINGS[0].rootLabel && BINDINGS[0].rootLabel.value) ? BINDINGS[0].rootLabel.value : "Scheme";
        
        let topConceptsXml = "";
        let conceptsXml = "";
        const processedConcepts = new Set();

        BINDINGS.forEach(function(b) {
            const cUri = b.child.value;
            const pUri = b.parent.value;

            if (allExportUris.has(cUri) && !processedConcepts.has(cUri)) {
                processedConcepts.add(cUri);

                const cb = document.querySelector('input[data-uri="' + cUri + '"]');
                const cleanLabel = cb ? cb.getAttribute('data-label') : ((b.childLabel && b.childLabel.value) ? b.childLabel.value.split(' (said to be the same)')[0] : cUri.split('/').pop());

                conceptsXml += '  <skos:Concept rdf:about="' + escapeXml(cUri) + '">\n    <skos:inScheme rdf:resource="' + escapeXml(schemeUri) + '"/>\n    <skos:prefLabel xml:lang="en">' + escapeXml(cleanLabel) + '</skos:prefLabel>\n';

                if (b.altLabels && b.altLabels.value) {
                    b.altLabels.value.split('||').forEach(function(alt) {
                        if (alt) conceptsXml += '    <skos:altLabel xml:lang="en">' + escapeXml(alt) + '</skos:altLabel>\n';
                    });
                }
                if (b.descriptions && b.descriptions.value) {
                    b.descriptions.value.split('||').forEach(function(desc) {
                        if (desc) conceptsXml += '    <skos:definition xml:lang="en">' + escapeXml(desc) + '</skos:definition>\n';
                    });
                }

                if (pUri === schemeUri) {
                    topConceptsXml += '    <skos:hasTopConcept rdf:resource="' + escapeXml(cUri) + '"/>\n';
                } else if (allExportUris.has(pUri)) {
                    conceptsXml += '    <skos:broader rdf:resource="' + escapeXml(pUri) + '"/>\n';
                }

                const narrowerSet = new Set();
                BINDINGS.forEach(function(sub) {
                    if (sub.parent.value === cUri && allExportUris.has(sub.child.value)) {
                        narrowerSet.add(sub.child.value);
                    }
                });
                narrowerSet.forEach(function(nUri) {
                    conceptsXml += '    <skos:narrower rdf:resource="' + escapeXml(nUri) + '"/>\n';
                });

                conceptsXml += '  </skos:Concept>\n\n';
            }
        });

        const xmlContent = '<?xml version="1.0" encoding="utf-8"?>\n' +
'<rdf:RDF\n' +
'  xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"\n' +
'  xmlns:skos="http://www.w3.org/2004/02/skos/core#"\n' +
'  xmlns:dcterms="http://purl.org/dc/terms/"\n' +
'>\n' +
'  <skos:ConceptScheme rdf:about="' + escapeXml(schemeUri) + '">\n' +
'    <dcterms:title xml:lang="en">' + escapeXml(schemeLabel) + '</dcterms:title>\n' +
topConceptsXml + '  </skos:ConceptScheme>\n\n' +
conceptsXml + '</rdf:RDF>';
        
        const filename = schemeLabel
            .normalize('NFKD').replace(/[\u0300-\u036f]/g, '')
            .replace(/[^a-z0-9]+/gi, '_')
            .replace(/^_+|_+$/g, '') || 'export';

        const blob = new Blob([xmlContent], { type: "application/rdf+xml;charset=utf-8;" });
        const downloadAnchor = document.createElement('a');
        downloadAnchor.href = URL.createObjectURL(blob);
        downloadAnchor.setAttribute("download", filename + ".rdf");
        document.body.appendChild(downloadAnchor);
        downloadAnchor.click();
        downloadAnchor.remove();
    });

    initDictionary(MY_QUERY);
});

mw.loader.using(['ext.uls.mediawiki'], function (){
    mw.uls.getFrequentLanguageList = function (){
        return ['en','pl','el','fr','de'];
    };
});

mw.hook('wikibase.entityPage.entityView.rendered').add(function(){
    mw.loader.using('wikibase').done(function(){
        var moreLabel = mw.msg('wikibase-entitytermsforlanguagelistview-more');

        $('a[href="#"]').filter(function(){
            return $(this).text().trim() === moreLabel;
        }).trigger('click');
    });
});

```
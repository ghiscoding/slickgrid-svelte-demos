<script lang="ts">
  import { ExcelExportService } from '@slickgrid-universal/excel-export';
  import { TextExportService } from '@slickgrid-universal/text-export';
  import {
    Filters,
    Formatters,
    SlickgridSvelte,
    type Column,
    type Formatter,
    type GridOption,
    type GridStateChange,
    type SlickgridSvelteInstance,
  } from 'slickgrid-svelte';
  import { tick } from 'svelte';

  import { localeFrench } from './locales/fr.js';

  const NB_ITEMS = 1500;
  const excelExportService = new ExcelExportService();
  const textExportService = new TextExportService();
  let showSubTitle = $state(true);
  let duplicateTitleHeaderCount = 1;
  let svelteGrid!: SlickgridSvelteInstance;

  const exportBooleanFormatter: Formatter = (_row, _cell, value) => {
    return value ? 'Vrai' : 'Faux';
  };

  const taskFormatter: Formatter = (_row, _cell, value) => {
    return value !== undefined ? `Titre ${value}` : '';
  };

  // create a custom translate Formatter (typically you would move that a separate file, for separation of concerns)
  const taskTranslateFormatter: Formatter = (_row, _cell, value, _columnDef, _dataContext, grid) => {
    const gridOptions = grid.getOptions() as GridOption;

    return gridOptions.i18n?.t('TASK_X', { x: value }) ?? '';
  };

  /* Define grid Options and Columns */
  let columns = $state.raw<Column[]>([
    { id: 'title', name: 'Titre', field: 'id', sortable: true, minWidth: 100, filterable: true, formatter: taskFormatter, params: { useFormatterOuputToFilter: true } },
    { id: 'description', name: 'Description', field: 'description', filterable: true, sortable: true, minWidth: 80 },
    {
      id: 'duration', name: 'Durée (jours)', field: 'duration', sortable: true,
      formatter: Formatters.percentCompleteBar,
      minWidth: 100,
      filterable: true,
      filter: { model: Filters.compoundSlider, operator: '>=' }
    },
    {
      id: 'start', name: 'Début', field: 'start', minWidth: 100,
      formatter: Formatters.dateIso, outputType: 'dateIso', type: 'date', exportWithFormatter: true,
      filterable: true, filter: { model: Filters.compoundDate }
    },
    {
      id: 'finish', name: 'Fin', field: 'finish',
      formatter: Formatters.dateIso, outputType: 'dateIso', type: 'date', exportWithFormatter: true,
      minWidth: 100, filterable: true, filter: { model: Filters.compoundDate }
    },
    {
      id: 'completedBool', name: 'Complétée', field: 'completedBool', minWidth: 100,
      sortable: true,
      formatter: Formatters.checkmarkMaterial,
      exportCustomFormatter: exportBooleanFormatter,
      filterable: true,
      filter: {
        collection: [{ value: true, label: 'Vrai' }, { value: false, label: 'Faux' }],
        model: Filters.multipleSelect,
      }
    }
  ]);

  const gridOptions: GridOption = {
    autoResize: {
      container: '#demo-container',
      rightPadding: 10
    },
    // use a Single Custom Locales set
    locale: 'fr', // this helps certain elements to know which locale to use, for example the Date Filter/Editor
    locales: localeFrench,
    enableAutoResize: true,
    enableExcelCopyBuffer: true,
    enableFiltering: true,
    checkboxSelector: {
      // you can toggle these 2 properties to show the "select all" checkbox in different location
      hideInFilterHeaderRow: false,
      hideInColumnTitleRow: true
    },
    enableCheckboxSelector: true,
    enableSelection: true,
    showCustomFooter: true, // display some metrics in the bottom custom footer
    customFooterOptions: {
      // optionally display some text on the left footer container
      // leftFooterText: 'custom text shown on left container',
      metricTexts: {
        // default text displayed in the metrics section on the right
        items: 'éléments',
        of: 'de',
        lastUpdate: 'dernière mise à jour',
      },
      dateFormat: 'YYYY-MM-DD, hh:mm a',
      hideTotalItemCount: false,
      hideLastUpdateTimestamp: false,
    },
    excelExportOptions: {
      // optionally pass a custom header to the Excel Sheet
      // a lot of the info can be found on Excel-Builder-Vanilla
      // https://ghiscoding.gitbook.io/excel-builder-vanilla/cookbook/fonts-and-colors
      customExcelHeader: (workbook, sheet) => {
        const stylesheet = workbook.getStyleSheet();
        const aFormatDefn = {
          'font': { 'size': 12, 'fontName': 'Calibri', 'bold': true, color: 'FF0000FF' }, // every color starts with FF, then regular HTML color
          'alignment': { 'wrapText': true }
        };
        const formatterId = stylesheet.createFormat(aFormatDefn);
        sheet.setRowInstructions(0, { height: 30 }); // change height of row 0

        // excel cells start with A1 which is upper left corner
        sheet.mergeCells('B1', 'D1');
        const cols = [];
        // push empty data on A1
        cols.push({ value: '' });
        // push data in B1 cell with metadata formatter
        cols.push({ value: 'Titre qui est suffisament long pour être coupé', metadata: { style: formatterId.id } });
        sheet.data.push(cols);
      }
    },
    enableExcelExport: true,
    enableTextExport: true,
    textExportOptions: {
      // set at the grid option level, meaning all column will evaluate the Formatter (when it has a Formatter defined)
      exportWithFormatter: true,
      sanitizeDataExport: true,
      menuFormats: ['csv', 'txt']
    },
    externalResources: [excelExportService, textExportService],
  };

  // mock some data
  const dataset = getData(NB_ITEMS);

  function getData(count: number) {
    // mock a dataset
    const tmpData: any[] = [];
    for (let i = 0; i < count; i++) {
      const randomYear = 2000 + Math.floor(Math.random() * 10);
      const randomMonth = Math.floor(Math.random() * 11);
      const randomDay = Math.floor(Math.random() * 29);

      tmpData[i] = {
        id: i,
        description: i % 5 ? 'desc ' + i : '🚀🦄 español', // also add some random to test NULL field
        duration: Math.round(Math.random() * 100) + '',
        start: new Date(randomYear, randomMonth, randomDay),
        finish: new Date(randomYear, randomMonth + 1, randomDay),
        completedBool: i % 5 === 0 ? true : false,
        completed: i % 5 === 0 ? 'TRUE' : 'FALSE',
      };
    }
    return tmpData;
  }

  function dynamicallyAddTitleHeader() {
    // you can dynamically add your column to your column definitions
    // and then use the spread operator [...cols] OR slice to force Svelte to review the changes
    const newCol: Column = {
      id: `title${duplicateTitleHeaderCount++}`,
      field: 'id',
      nameKey: 'TITLE',
      formatter: taskTranslateFormatter,
      sortable: true,
      minWidth: 100,
      filterable: true,
      params: { useFormatterOuputToFilter: true },
    };
    columns = [...columns, newCol];
  }

  function exportToExcel() {
    excelExportService.exportToExcel({
      filename: 'Export',
      format: 'xlsx',
    });
  }

  function exportToFile(type = 'csv') {
    textExportService.exportToFile({
      delimiter: type === 'csv' ? ',' : '\t',
      filename: 'myExport',
      format: type === 'csv' ? 'csv' : 'txt',
    });
  }

  /** Dispatched event of a Grid State Changed event */
  function gridStateChanged(gridStateChanges: GridStateChange) {
    console.log('Grid State changed:: ', gridStateChanges);
    console.log('Grid State changed:: ', gridStateChanges.change);
  }

  async function toggleSubTitle() {
    showSubTitle = !showSubTitle;
    await tick();
    svelteGrid.resizerService.resizeGrid();
  }
</script>

<h2>
  Example 12: Localization (i18n)
  <span class="float-end">
    <a
      style="font-size: 18px"
      target="_blank"
      rel="noreferrer"
      href="https://github.com/ghiscoding/slickgrid-universal/blob/master/demos/svelte/src/Example12.svelte"
    >
      <span class="mdi mdi-link-variant"></span> code
    </a>
  </span>
  <button
    class="ms-2 btn btn-outline-secondary btn-sm btn-icon"
    type="button"
    aria-label="Toggle example sub-title details"
    data-test="toggle-subtitle"
    onclick={toggleSubTitle}
  >
    <span class="mdi mdi-information-outline" title="Toggle example sub-title details"></span>
  </button>
</h2>

{#if showSubTitle}
  <div class="subtitle">
    Support multiple locales with the i18next plugin, following these steps. Take a look at the (<a
      href="https://ghiscoding.gitbook.io/svelte-slickgrid/localization/localization"
      target="_blank"
      rel="noreferrer"
    >
      Wiki documentation
    </a>)
    <ol class="small">
      <li>You first need to "enableTranslate" in the Grid Options</li>
      <li>In the Column Definitions, you have following options</li>
      <ul>
        <li>To translate a header title, use "nameKey" with a translate key (nameKey: 'TITLE')</li>
        <li>For the cell values, you need to use a Formatter, there's 2 ways of doing it</li>
        <ul>
          <li>formatter: myCustomTranslateFormatter <b>&lt;= "Title" column uses it</b></li>
          <li>formatter: Formatters.translate <b>&lt;= "Completed" column uses it</b></li>
        </ul>
      </ul>
      <li>For date localization, you need to create your own custom formatter.</li>
      <ul>
        <li>You can easily implement logic to switch between Formatters "dateIso" or "dateUs", depending on current locale.</li>
      </ul>
      <li>For the Select (dropdown) filter, you can fill in the "labelKey" property, if found it will use it, else it will use "label"</li>
      <ul>
        <li>
          What if your select options have totally different value/label pair? In this case, you can use the
          <b>customStructure: &#123; label: 'customLabel', value: 'customValue'&#125;</b> to change the property name(s) to use.'
        </li>
        <li>What if you want to use "customStructure" and translation? Simply pass this flag <b>enableTranslateLabel: true</b></li>
        <li>
          More info on the Select Filter
          <a
            href="https://ghiscoding.gitbook.io/svelte-slickgrid/column-functionalities/filters/select-filter"
            target="_blank"
            rel="noreferrer"
          >
            Wiki page
          </a>
        </li>
      </ul>
      <li>
        For more info about "Download to File", read the
        <a href="https://ghiscoding.gitbook.io/svelte-slickgrid/grid-functionalities/export-to-excel" target="_blank" rel="noreferrer">
          Wiki page
        </a>
      </li>
    </ol>
  </div>
{/if}

<hr />

<div class="row">
  <div class="col-sm-12">
    <span style="margin-left: 20px">
      <button class="btn btn-outline-secondary btn-sm btn-icon" onclick={() => exportToFile('csv')}>
        <i class="mdi mdi-download"></i>
        Download to CSV
      </button>
      <button class="btn btn-outline-secondary btn-sm btn-icon mx-1" onclick={() => exportToFile('txt')}>
        <i class="mdi mdi-download"></i>
        Download to Text
      </button>
      <button class="btn btn-outline-secondary btn-sm btn-icon" onclick={exportToExcel}>
        <i class="mdi mdi-file-excel-outline text-success"></i>
        Download to Excel
      </button>
    </span>
    <span style="margin-left: 10px">
      <button class="btn btn-outline-secondary btn-sm btn-icon" onclick={dynamicallyAddTitleHeader}>
        <i class="mdi mdi-shape-square-plus"></i>
        Dynamically Duplicate Title Column
      </button>
    </span>
  </div>
</div>

<SlickgridSvelte
  gridId="grid12"
  {columns}
  options={gridOptions}
  {dataset}
  onGridStateChanged={(event) => gridStateChanged(event.detail)}
  onSvelteGridCreated={(instance) => (svelteGrid = instance)}
/>

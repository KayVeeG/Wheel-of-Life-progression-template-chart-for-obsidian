# Wheel-of-Life-progression-template-chart-for-obsidian



```
<%*
// =============================================================================
// WHEEL OF LIFE QUARTERLY CHART - TEMPLATER VERSION
// =============================================================================
// This template generates a quarterly progression chart for "Wheel of Life" data
// from weekly notes, with optional monthly averages overlay.
//
// REQUIREMENTS:
// - Weekly notes tagged with #type/weekly-note
// - Monthly notes tagged with #type/monthly-note  
// - wheelOfLife data structure in weekly notes
// - Quarterly file naming: YYYY-Q#.md
// =============================================================================

// CONFIGURATION - Customize these values for your setup
const CONFIG = {
  // Chart styling
  chartHeight: '500px',
  chartType: tp.frontmatter.quarterlyWheelOfLifeCategoryChartType || 'line',
  
  // Note tags - adjust if you use different tags
  weeklyNoteTag: '#type/weekly-note',
  monthlyNoteTag: '#type/monthly-note',
  
  // Categories - customize for your wheel of life areas
  categories: [
    { key: 'spiritual', label: 'Spiritual', color: 'rgba(103,58,183,0.9)' },
    { key: 'career', label: 'Career/Work', color: 'rgba(3,169,244,0.9)' },
    { key: 'relationships', label: 'Love/Relationships', color: 'rgba(233,30,99,0.9)' },
    { key: 'health', label: 'Health/Fitness', color: 'rgba(0,150,136,0.9)' },
    { key: 'growth', label: 'Personal Growth', color: 'rgba(255,87,34,0.9)' },
    { key: 'recreation', label: 'Fun/Recreation', color: 'rgba(255,193,7,0.9)' },
    { key: 'social', label: 'Social', color: 'rgba(156,39,176,0.9)' },
    { key: 'finance', label: 'Finance', color: 'rgba(76,175,80,0.9)' }
  ]
};

// UTILITY FUNCTIONS
function parseQuarterFromFilename(filename) {
  const quarterMatch = filename.match(/(\d{4})-Q(\d)/);
  if (!quarterMatch) return null;
  
  return {
    year: parseInt(quarterMatch[1]),
    quarter: parseInt(quarterMatch[2])
  };
}

function getQuarterDateRange(year, quarter) {
  const quarterStart = moment().year(year).quarter(quarter).startOf('quarter');
  const quarterEnd = moment().year(year).quarter(quarter).endOf('quarter');
  return { start: quarterStart, end: quarterEnd };
}

function getNotesInDateRange(tag, startDate, endDate) {
  return app.vault.getMarkdownFiles()
    .filter(file => {
      // Check if file has the required tag
      const cache = app.metadataCache.getFileCache(file);
      if (!cache?.tags?.some(t => t.tag === tag)) return false;
      
      // Check if file has a date (either in filename or frontmatter)
      const fileDate = getNoteDateFromFile(file);
      if (!fileDate) return false;
      
      return fileDate.isSameOrAfter(startDate, 'day') && 
             fileDate.isSameOrBefore(endDate, 'day');
    })
    .sort((a, b) => {
      const dateA = getNoteDateFromFile(a);
      const dateB = getNoteDateFromFile(b);
      return dateA.valueOf() - dateB.valueOf();
    });
}

function getNoteDateFromFile(file) {
  // Try to get date from filename first (YYYY-MM-DD format)
  const fileNameMatch = file.name.match(/(\d{4}-\d{2}-\d{2})/);
  if (fileNameMatch) {
    return moment(fileNameMatch[1]);
  }
  
  // Try to get date from frontmatter
  const cache = app.metadataCache.getFileCache(file);
  if (cache?.frontmatter?.date) {
    return moment(cache.frontmatter.date);
  }
  
  return null;
}

async function getWheelOfLifeData(file) {
  const content = await app.vault.read(file);
  const cache = app.metadataCache.getFileCache(file);
  
  // Try frontmatter first
  if (cache?.frontmatter?.wheelOfLife) {
    return cache.frontmatter.wheelOfLife;
  }
  
  // Try to parse from content (look for wheelOfLife object)
  const wheelMatch = content.match(/wheelOfLife:\s*\{([^}]+)\}/);
  if (wheelMatch) {
    try {
      const wheelData = {};
      const entries = wheelMatch[1].split(',');
      entries.forEach(entry => {
        const [key, value] = entry.split(':').map(s => s.trim());
        wheelData[key] = parseInt(value);
      });
      return wheelData;
    } catch (e) {
      console.warn(`Failed to parse wheelOfLife data from ${file.name}`);
    }
  }
  
  return null;
}

function calculateMonthlyAverages(weeklyNotes, categories) {
  const monthlyData = {};
  
  // Group weeks by month
  weeklyNotes.forEach(async (file) => {
    const date = getNoteDateFromFile(file);
    const monthKey = date.format('YYYY-MM');
    
    if (!monthlyData[monthKey]) {
      monthlyData[monthKey] = { weeks: [], averages: {} };
    }
    
    monthlyData[monthKey].weeks.push(file);
  });
  
  // Calculate averages for each month
  for (const monthKey in monthlyData) {
    const monthData = monthlyData[monthKey];
    
    categories.forEach(category => {
      let sum = 0;
      let count = 0;
      
      monthData.weeks.forEach(async (file) => {
        const wheelData = await getWheelOfLifeData(file);
        if (wheelData && typeof wheelData[category.key] === 'number') {
          sum += wheelData[category.key];
          count++;
        }
      });
      
      monthData.averages[category.key] = count > 0 ? sum / count : null;
    });
  }
  
  return monthlyData;
}

function createChartConfig(weeklyNotes, monthlyData, categories, chartType) {
  const weekLabels = weeklyNotes.map(file => {
    const weekMatch = file.name.match(/W(\d+)/);
    return weekMatch ? `W${weekMatch[1]}` : file.name.replace('.md', '');
  });
  
  const datasets = [];
  
  // Weekly data datasets
  categories.forEach((category, index) => {
    const weeklyData = weeklyNotes.map(async (file) => {
      const wheelData = await getWheelOfLifeData(file);
      return wheelData && typeof wheelData[category.key] === 'number' 
        ? wheelData[category.key] 
        : null;
    });
    
    datasets.push({
      label: category.label,
      data: weeklyData,
      borderColor: category.color,
      backgroundColor: category.color.replace('0.9', '0.4'),
      borderWidth: 3,
      tension: 0.3,
      fill: false,
      pointRadius: 6,
      pointHoverRadius: 8,
      hidden: true
    });
  });
  
  // Monthly average datasets
  categories.forEach((category, index) => {
    const monthlyAverageData = weeklyNotes.map(file => {
      const date = getNoteDateFromFile(file);
      const monthKey = date.format('YYYY-MM');
      return monthlyData[monthKey]?.averages[category.key] || null;
    });
    
    datasets.push({
      label: `${category.label} Monthly Avg`,
      data: monthlyAverageData,
      borderColor: category.color.replace('0.9', '0.4'),
      backgroundColor: 'transparent',
      borderWidth: 2,
      borderDash: [5, 5],
      tension: 0,
      fill: false,
      pointRadius: 0,
      pointHoverRadius: 0,
      hidden: true
    });
  });
  
  return {
    type: chartType,
    data: {
      labels: weekLabels,
      datasets: datasets
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      interaction: {
        mode: 'index',
        intersect: false
      },
      plugins: {
        tooltip: {
          callbacks: {
            title: (context) => {
              const idx = context[0].dataIndex;
              return idx >= 0 && idx < weeklyNotes.length 
                ? weeklyNotes[idx].name 
                : '';
            }
          }
        },
        legend: {
          position: 'bottom',
          align: 'start',
          labels: {
            usePointStyle: true,
            padding: 15,
            font: { family: "'Inter', sans-serif", size: 11 },
            color: 'white'
          }
        }
      },
      scales: {
        y: {
          beginAtZero: true,
          max: 10,
          grid: { color: 'rgba(255, 255, 255, 0.1)' },
          ticks: {
            color: 'white',
            font: { family: "'Inter', sans-serif", size: 11 }
          },
          title: {
            display: true,
            text: 'Rating (1-10)',
            color: 'white',
            font: { family: "'Inter', sans-serif", size: 12 }
          }
        },
        x: {
          grid: { display: false },
          ticks: {
            color: 'white',
            font: { family: "'Inter', sans-serif", size: 11 }
          }
        }
      },
      onClick: (e, elements) => {
        if (elements.length > 0) {
          const index = elements[0].index;
          if (index >= 0 && index < weeklyNotes.length) {
            const file = weeklyNotes[index];
            app.workspace.openLinkText(file.name, '', true);
          }
        }
      },
      animation: {
        duration: 1000,
        easing: 'easeOutQuad'
      }
    }
  };
}

// MAIN EXECUTION
try {
  // Parse quarter information from current file
  const currentFileName = tp.file.title;
  const quarterInfo = parseQuarterFromFilename(currentFileName);
  
  if (!quarterInfo) {
    tR += "⚠️ Error: Couldn't parse quarter from filename. Expected format: YYYY-Q#";
  } else {
    const { year, quarter } = quarterInfo;
    const dateRange = getQuarterDateRange(year, quarter);
    
    // Get weekly and monthly notes for this quarter
    const weeklyNotes = getNotesInDateRange(CONFIG.weeklyNoteTag, dateRange.start, dateRange.end);
    const monthlyNotes = getNotesInDateRange(CONFIG.monthlyNoteTag, dateRange.start, dateRange.end);
    
    if (weeklyNotes.length === 0) {
      tR += "⚠️ No weekly notes found for this quarter. Please create weekly notes with wheelOfLife data first.";
    } else {
      // Calculate monthly averages
      const monthlyData = calculateMonthlyAverages(weeklyNotes, CONFIG.categories);
      
      // Create chart configuration
      const chartConfig = createChartConfig(weeklyNotes, monthlyData, CONFIG.categories, CONFIG.chartType);
      
      // Generate chart HTML
      const chartId = `wheel-of-life-${year}-q${quarter}`;
      
      tR += `### Wheel of Life Progression - ${year} Q${quarter}

<div id="${chartId}" style="height: ${CONFIG.chartHeight}; margin: 30px 0 40px 0;"></div>

<script>
(function() {
  const chartConfig = ${JSON.stringify(chartConfig, null, 2)};
  
  // Wait for Chart.js to be available
  function renderChart() {
    if (typeof Chart !== 'undefined') {
      const ctx = document.getElementById('${chartId}');
      if (ctx) {
        new Chart(ctx, chartConfig);
      }
    } else {
      setTimeout(renderChart, 100);
    }
  }
  
  renderChart();
})();
</script>

**Chart Instructions:**
- All categories are hidden by default - click legend items to show/hide
- Click on data points to open corresponding weekly notes
- Monthly averages are shown as dashed lines when enabled
- Found ${weeklyNotes.length} weekly notes for this quarter`;
    }
  }
} catch (error) {
  tR += `⚠️ Error generating chart: ${error.message}`;
  console.error("Wheel of Life chart error:", error);
}
_%>
```

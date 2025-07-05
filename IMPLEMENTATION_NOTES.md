# M4B-Metadata: Implementation Notes

## Project Overview

This document outlines the creation of an enhanced version of [Bragibooks](https://github.com/djdembeck/bragibooks) with specific modifications to ensure full compatibility with [Audiobookshelf](https://github.com/advplyr/audiobookshelf) metadata requirements.

## Key Requirements Addressed

Based on research into Audiobookshelf's metadata handling, the following requirements were identified and implemented:

### 1. ASIN Preservation
- **Problem**: Original bragibooks didn't save ASIN numbers in M4B metadata
- **Solution**: Added ASIN preservation in multiple metadata fields:
  - Custom `ASIN` tag for direct Audiobookshelf access
  - ASIN included in comment field for broader compatibility
  - Post-processing validation to ensure ASIN is properly embedded

### 2. Author Format Compatibility
- **Problem**: Author metadata formatting needed to match Audiobookshelf expectations
- **Solution**: Enhanced author handling:
  - Multiple authors formatted as comma-separated lists
  - Proper mapping to `albumartist` field
  - Maintains original author data structure in database

### 3. Narrator Integration
- **Problem**: Narrator information needed proper embedding for Audiobookshelf
- **Solution**: Enhanced narrator support:
  - Multiple narrators supported and comma-separated
  - Mapped to `artist` field (standard for audiobooks)
  - Clear distinction maintained between authors and narrators

## Technical Implementation

### Project Structure

```
m4b-metadata/
├── utils/
│   ├── audiobookshelf_metadata.py  # New: Audiobookshelf metadata writer
│   ├── merge.py                    # Modified: Enhanced with ABS compatibility
│   ├── search_tools.py            # Unchanged
│   └── region_tools.py            # Unchanged
├── importer/                      # Original Django app structure
├── bragibooks_proj/              # Original Django project structure
└── docker/                       # Original Docker configuration
```

### Key Files Created/Modified

#### 1. `utils/audiobookshelf_metadata.py` (New)
**Purpose**: Dedicated module for Audiobookshelf-compatible metadata handling

**Key Features**:
- `AudiobookshelfMetadataWriter` class for enhanced metadata operations
- `prepare_audiobookshelf_metadata_args()` - Formats metadata for m4b-tool
- `apply_audiobookshelf_metadata()` - Applies metadata to M4B files
- `validate_audiobookshelf_metadata()` - Validates metadata after processing
- Text cleaning and sanitization for metadata fields

**Metadata Format**:
```python
metadata_args = [
    f"--name={combined_title}",           # Full title with subtitle
    f"--album={title}",                   # Base title
    f"--artist={narrators_formatted}",   # Comma-separated narrators
    f"--albumartist={authors_formatted}", # Comma-separated authors
    f"--custom-tag=ASIN:{asin}",         # ASIN as custom tag
    f"--comment=ASIN: {asin}",           # ASIN in comment for compatibility
    # ... additional fields
]
```

#### 2. `utils/merge.py` (Enhanced)
**Purpose**: Modified to integrate Audiobookshelf metadata processing

**Key Changes**:
- Added `AudiobookshelfM4bMerge` class extending original `M4bMerge`
- Enhanced `prepare_command_args()` to use new metadata writer
- Added `apply_post_processing_metadata()` for additional metadata application
- Modified `run_m4b_merge()` to use enhanced class
- Added comprehensive logging for metadata operations

**Processing Flow**:
1. Original m4b-merge processing
2. Enhanced metadata preparation using `AudiobookshelfMetadataWriter`
3. Post-processing metadata application
4. Validation of applied metadata

### Compatibility Strategy

The implementation maintains full backward compatibility with the original bragibooks while adding Audiobookshelf enhancements:

#### Original Functionality Preserved
- All original Django models and database structure
- Original web interface and user experience
- Original m4b-merge integration
- Original Docker configuration

#### Audiobookshelf Enhancements Added
- ASIN preservation in metadata
- Enhanced author/narrator formatting
- Custom metadata tags for better Audiobookshelf integration
- Metadata validation and error handling
- Debug logging for troubleshooting

## Research Findings

### Audiobookshelf Metadata Requirements

Based on analysis of Audiobookshelf source code and API documentation:

1. **Author Field**: Expects comma-separated author names in `albumartist` tag
2. **Narrator Field**: Expects comma-separated narrator names in `artist` tag
3. **ASIN Support**: Can read ASIN from custom tags or comment fields
4. **Title Handling**: Supports both simple title and title with subtitle
5. **Series Information**: Reads series name and position from standard tags

### m4b-tool Integration

Research into m4b-tool capabilities revealed:

1. **Custom Tags**: Supports `--custom-tag=KEY:VALUE` format
2. **Metadata Override**: Can override existing metadata with `--force` flag
3. **Multiple Authors/Narrators**: Handles comma-separated values properly
4. **Validation**: Can validate metadata after application

### Compatibility Testing Considerations

For proper testing, the implementation should be validated with:

1. **Multiple Authors**: Books with 2+ authors to test comma separation
2. **Multiple Narrators**: Books with 2+ narrators to test formatting
3. **Series Books**: Books with series information to test series handling
4. **ASIN Validation**: Various ASIN formats to ensure proper embedding
5. **Audiobookshelf Import**: Actual import testing in Audiobookshelf

## Deployment Recommendations

### Docker Deployment (Recommended)

```yaml
version: '3'
services:
  m4b-metadata:
    build: .
    container_name: m4b-metadata
    environment:
      - LOG_LEVEL=INFO
      - DEBUG=False
      - UID=1000
      - GID=1000
    volumes:
      - ./config:/config
      - /path/to/audiobooks/input:/input
      - /path/to/audiobooks/output:/output
      - /path/to/processed:/done
    ports:
      - "8000:8000"
    restart: unless-stopped
```

### Environment Configuration

| Variable | Recommended Value | Purpose |
|----------|------------------|---------|
| `LOG_LEVEL` | `INFO` | Provides detailed processing information |
| `DEBUG` | `False` | Production setting |
| `UID`/`GID` | User's UID/GID | Proper file permissions |
| `CELERY_WORKERS` | `1-2` | Processing concurrency |

## Validation and Testing

### Metadata Validation Process

1. **Pre-Processing**: Validate ASIN format and availability
2. **During Processing**: Log metadata argument construction
3. **Post-Processing**: Validate embedded metadata using ffprobe
4. **Audiobookshelf Import**: Test actual import and metadata display

### Test Cases

1. **Single Author/Narrator**: Basic functionality test
2. **Multiple Authors**: "Author 1, Author 2" format test
3. **Multiple Narrators**: "Narrator 1, Narrator 2" format test
4. **Series Books**: Series name and position embedding test
5. **ASIN Embedding**: Custom tag and comment field validation
6. **Special Characters**: Unicode and special character handling
7. **Long Descriptions**: Text length and sanitization testing

## Troubleshooting Guide

### Common Issues and Solutions

#### ASIN Not Appearing in Audiobookshelf
**Symptoms**: ASIN not visible in Audiobookshelf after import
**Solutions**:
1. Check logs for metadata application errors
2. Validate ASIN format (should be 10 characters)
3. Use ffprobe to verify ASIN in metadata
4. Re-scan library in Audiobookshelf

#### Authors Not Properly Displayed
**Symptoms**: Authors missing or improperly formatted
**Solutions**:
1. Verify comma-separated format in albumartist field
2. Check for special characters in author names
3. Validate metadata with ffprobe
4. Ensure proper encoding in metadata

#### Narrators Missing
**Symptoms**: Narrator information not displayed
**Solutions**:
1. Verify narrator data in original Audible metadata
2. Check artist field contains narrator names
3. Validate comma separation for multiple narrators
4. Check Audiobookshelf narrator parsing

### Debug Commands

```bash
# Check metadata in processed M4B
ffprobe -v quiet -print_format json -show_format file.m4b

# Validate ASIN embedding
ffprobe -v quiet -show_entries format_tags=ASIN file.m4b

# Check author/narrator formatting
ffprobe -v quiet -show_entries format_tags=artist,album_artist file.m4b

# Enable debug logging
export LOG_LEVEL=DEBUG
```

## Future Enhancements

### Potential Improvements

1. **Enhanced Series Support**: Better handling of series with subseries
2. **Publisher Information**: Enhanced publisher metadata embedding
3. **Genre Optimization**: Better genre formatting for Audiobookshelf
4. **Batch Processing**: Optimized metadata application for multiple files
5. **Metadata Templates**: Customizable metadata templates for different use cases

### Integration Opportunities

1. **Audiobookshelf API**: Direct integration with Audiobookshelf API
2. **Metadata Sync**: Bidirectional metadata synchronization
3. **Library Management**: Enhanced library organization features
4. **Backup/Restore**: Metadata backup and restoration capabilities

## Conclusion

The enhanced M4B-Metadata implementation successfully addresses the key metadata compatibility requirements for Audiobookshelf while maintaining full backward compatibility with the original project. The modular design allows for easy maintenance and future enhancements while providing robust metadata handling specifically tailored for Audiobookshelf users.

The implementation provides:

✅ **ASIN Preservation**: Embedded in multiple metadata fields
✅ **Author Compatibility**: Proper comma-separated formatting
✅ **Narrator Support**: Enhanced narrator metadata embedding
✅ **Validation**: Built-in metadata validation and error handling
✅ **Backward Compatibility**: Maintains all original functionality
✅ **Documentation**: Comprehensive setup and troubleshooting guides

This enhanced version ensures that audiobooks processed through bragibooks will integrate seamlessly with Audiobookshelf, providing users with properly formatted metadata for optimal library management and browsing experience.